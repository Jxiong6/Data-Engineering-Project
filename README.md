# Data-Engineering-Project

- Extract the data from a CSV file 
- Store the data in bigquery
- Interacts with DBT (in airflow)
- Create a dashboard with metabase



## The process

First we need to use airflow to create a task that loads the CSV file into GCS. 

Then we will run data quality checks by using SODA to make sure that the raw data is correct.(`Quality Checks`)

We will run another task in order to shape the raw data so we obtain the fact table and the dimension tables by using DBT and airflow. (`Run dbt models to transform`)

After we obtain the fact table and dimension table we run data quality checks again we want to make sure that the Transformations  are correct .(`Quality Checks`)

And we run all the models in order to get analytics from the transformed  data basing DBT. (`Run dbt models for report`)

And we run final data quality to make sure the analytics are correct.(`Quality Checks`)

Build the dashboard and get insights from the data.(`Metabase`)

### Start the airflow project

First we need to install the astro 

```bash
winget install -e --id Astronomer.Astro -v 1.28.0
```

And then we init our project

```bash
astro dev init     
```

If  you want to run airflow 

```bash
astro dev start
```

And we can check in `localhost:8080`

### Download the data

Click the link :https://www.kaggle.com/datasets/tunguz/online-retail, and the download the dataset. Create a new folder `dataset` under `include`. And put the csv file in it.

### Ingest Raw Dataset into GCS

First we go to GCP and then create a new project `online retail`.

And we go to  Google cloud storage -> buckets. We need to create a bucket to store.(In this part, we can use `terraform` to create automatically )

The next step is to create a series second as we want to interact with this bucket from airflow.

So we go to IAM  -> service accounts and create a service account `airflow-online-retail`. And include `cloud storage-storage admin` and `bigquery`

And then click the second service and go to the `KEYS`, then click `ADD KEY` to create new key in JSON type. Once we have the Json file, we go back to the vscode. Create a new folder `gcp` under include. And put the json file in it.

And now we go to `requirements.txt` and add the following package which is Apache airflow providers.

```bash 
apache-airflow-providers-google==10.3.0
```

and we restart the airflow 

```bash
astro dev restart
```

Finally the last step is to create a connection in airflow that will use the `service_account.json` file. We go to the airflow UI, go to Admin->connections to create a new connection. First name it, and for the connection Type we choose Google cloud. For the `Keyfile Path`, we go the vscode, use the command `astro dev bash` , and then `ls` to find all the files and folders. Then we go to include-> gcp and use`pwd` to check the path:`/usr/local/airflow/include/gcp/service_account.json` And back to the airflow UI, and fill in the path /usr/local/airflow/include/gcp/service_account.json. 

We want to use the test function, so we go back to vscode then add `AIRFLOW_CORE_TEST_CONNECTION=enabled` in file .env  and restart it. 

### Create the data pipeline

#### upload csv to gcs

First we create a new file `retail.py` in dags

```python
from airflow.decorators import dag, task
from datetime import datetime

from airflow.providers.google.cloud.transfers.local_to_gcs import LocalFilesystemToGCSOperator

@dag(
    start_date=datetime(2024, 8, 14),
    schedule=None,
    catchup=False,
    tags=['retail'],
)
def retail():

    upload_csv_to_gcs = LocalFilesystemToGCSOperator(
        task_id='upload_csv_to_gcs',
        src='/usr/local/airflow/include/dataset/Online_Retail.csv',
        dst='raw/online_retail.csv',
        bucket='jiahui_online_retail',
        gcp_conn_id='gcp',
        mime_type='text/csv',
    )

retail()
```



For these libraries, we can search in `registry.astronomer.io`

In this case we want to test upload CSV to GCS and for this we can go to our terminal type 

```bash
astro dev bash 
```

and then type the following to test.

```bash
airflow tasks test <dag name> <task ID> <execution date>
```

like: 

```bash
airflow tasks test retail upload_csv_to_gcs 2024-08-14
```

And now we have the csv file in the bucket of GCS

#### load the CSV file as a BigQuery table

We also want to load this CSV file as a BigQuery table and for this we need to create two tasks.

`The first task` to implement is to create a dataset in BigQuery.

##### Create an empty Dataset(schema equivalent)

```python
from airflow.providers.google.cloud.operators.bigquery import BigQueryCreateEmptyDatasetOperator

create_retail_dataset = BigQueryCreateEmptyDatasetOperator(
        task_id='create_retail_dataset',
        dataset_id='retail',
        gcp_conn_id='gcp',
    )
```

And then still the test 

```bash
astro dev bash
airflow tasks test retail create_retail_dataset 2024-08-14
```

Now we can find `retail` in Bigquery of GCP.

`The second task` to implement is the task that it is in charge of uploading the CSV file from the bucket into a bigquery table that will be under the retail dataset.

The way we want to use is `Astro SDK`,

```python
from astro import sql as sql
from astro.files import File
from astro.sql.table import Table, Metadata
from astro.constants import FileType

    gcs_to_raw = aql.load_file(
        task_id='gcs_to_raw',
        input_file=File(
            'gs://jiahui_online_retail/raw/online_retail.csv',
            conn_id='gcp',
            filetype=FileType.CSV,
        ),
        output_table=Table(
            name='raw_invoices',
            conn_id='gcp',
            metadata=Metadata(schema='retail')
        ),
        use_native_support=False,
    )
```

And test it :

```bash
airflow tasks test retail gcs_to_raw 2024-08-14
```

Now we have uploaded our data into BiuQuery.



#### Quality Checks

now it's time to verify the raw data and for that we will use Soda.

First we need to install the soda python package in the requirements.txt

```bash
soda-core-bigquery==3.0.45
```

and create a new folder`soda` in `include`. And with a file `configuration.yml` file. This file will contain the connection to bigquery

```bash
-- include/soda/configuration.yml
data_source retail:
  type: bigquery
  connection:
    account_info_json_path: /usr/local/airflow/include/gcp/service_account.json
    auth_scopes:
    - https://www.googleapis.com/auth/bigquery
    - https://www.googleapis.com/auth/cloud-platform
    - https://www.googleapis.com/auth/drive
    project_id: 'online-retail-432615'
    dataset: retail
```

project_id is found in GCP

And then  restart 

```bash
astro dev restart
```

- Create an API → Profile → API Keys → Create API → Copy API in `configuration.yml`

And Test the connection

```bash
astro dev bash
soda test-connection -d retail -c include/soda/configuration.yml
```



##### Create the first test

Create a new folder `checks` in `soda`. And a folder `sources` in `checks`. And a new file `raw_invoices.yml` in `sources` .

```yml
checks for raw_invoices:
  - schema:
      fail:
        when required column missing: [InvoiceNo, StockCode, Quantity, InvoiceDate, UnitPrice, CustomerID, Country]
        when wrong column type:
          InvoiceNo: string
          StockCode: string
          Quantity: integer
          InvoiceDate: string
          UnitPrice: float64
          CustomerID: float64
          Country: string
```

Run the quality check

make sure you are in the container

```bash
soda scan -d retail -c include/soda/configuration.yml include/soda/checks/sources/raw_invoices.yml
```

Let's Create the check function in `retail.py`

First we creat the virtual env in `Dockfile`

```dockerfile
# install soda into a virtual environment
RUN python -m venv soda_venv && source soda_venv/bin/activate && \
    pip install --no-cache-dir soda-core-bigquery==3.0.45 &&\
    pip install --no-cache-dir soda-core-scientific==3.0.45 && deactivate
```

and in the `retail.py`

``` python
    @task.external_python(python='/usr/local/airflow/soda_venv/bin/python')
    def check_load(scan_name='check_load', checks_subpath='sources'):
        from include.soda.checks.sources.check_function import check

        return check(scan_name, checks_subpath)
    
    check_load()
```

And create a `check_function.py` in sources

```python
# include/soda/check_function.py
def check(scan_name, checks_subpath=None, data_source='retail', project_root='include'):
    from soda.scan import Scan

    print('Running Soda Scan ...')
    config_file = f'{project_root}/soda/configuration.yml'
    checks_path = f'{project_root}/soda/checks'

    if checks_subpath:
        checks_path += f'/{checks_subpath}'

    scan = Scan()
    scan.set_verbose()
    scan.add_configuration_yaml_file(config_file)
    scan.set_data_source_name(data_source)
    scan.add_sodacl_yaml_files(checks_path)
    scan.set_scan_definition_name(scan_name)

    result = scan.execute()
    print(scan.get_logs_text())

    if result != 0:
        raise ValueError('Soda Scan failed')

    return result
```

And then 

```bash
astro dev restart
astro dev bash 
airflow tasks test retail check_load 2024-08-15
```

works!

### Run dbt models to transform

First Install Cosmos -DBT

- In requirements.txt

  ```javascript
  // REMOVE apache-airflow-providers-google==10.3.0
  // REMOVE soda-core-bigquery==3.0.45
  astronomer-cosmos[dbt-bigquery]==1.0.3 // install google + cosmos + dbt
  protobuf==3.20.0
  ```

- In env

  ```bash
  PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python
  ```

- In Dockerfile

  ```dockerfile
  # install dbt into a virtual environment
  RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
      pip install --no-cache-dir dbt-bigquery==1.5.3 && deactivate
  ```

- restart

  ```bash
  astro dev restart
  ```



Create a new folder include/dbt

and a new file dbt/profiles.yml

```yaml
# profiles.yml

retail:
 target: dev
 outputs:
  dev:
    type: bigquery
    method: service-account
    keyfile: /usr/local/airflow/include/gcp/service_account.json
    project: online-retail-432615
    dataset: retail
    threads: 1
    timeout_seconds: 300
    location: US
```

change the project_id, and make sure the same location as it shows in dataset.

and dbt/dbt_project.yml

```yaml
# dbt_project.yml

name: 'retail'

profile: 'retail'

models:
  retail:
    materialized: table
```

and dbt/packages.yml

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

and then create a new folder `models`in dbt, and a folder `sources` in models. And a new file `sources.yml` in `sources`.

```yml
# sources.yml
version: 2

sources:
  - name: retail
    database: online-retail-432615 # Put the project id!

    tables:
      - name: raw_invoices
			- name: country
```

we need to create the table `country`

copy the code

```sql
CREATE TABLE IF NOT EXISTS `retail.country` (
  `id` INT NOT NULL,
  `iso` STRING NOT NULL,
  `name` STRING NOT NULL,
  `nicename` STRING NOT NULL,
  `iso3` STRING DEFAULT NULL,
  `numcode` INT DEFAULT NULL,
  `phonecode` INT NOT NULL,
);

--
-- Dumping data for table `country`
--

INSERT INTO `retail.country` (`id`, `iso`, `name`, `nicename`, `iso3`, `numcode`, `phonecode`) VALUES
(1, 'AF', 'AFGHANISTAN', 'Afghanistan', 'AFG', 4, 93),
(2, 'AL', 'ALBANIA', 'Albania', 'ALB', 8, 355),
(3, 'DZ', 'ALGERIA', 'Algeria', 'DZA', 12, 213),
(4, 'AS', 'AMERICAN SAMOA', 'American Samoa', 'ASM', 16, 1684),
(5, 'AD', 'ANDORRA', 'Andorra', 'AND', 20, 376),
(6, 'AO', 'ANGOLA', 'Angola', 'AGO', 24, 244),
(7, 'AI', 'ANGUILLA', 'Anguilla', 'AIA', 660, 1264),
(8, 'AQ', 'ANTARCTICA', 'Antarctica', NULL, NULL, 0),
(9, 'AG', 'ANTIGUA AND BARBUDA', 'Antigua and Barbuda', 'ATG', 28, 1268),
(10, 'AR', 'ARGENTINA', 'Argentina', 'ARG', 32, 54),
(11, 'AM', 'ARMENIA', 'Armenia', 'ARM', 51, 374),
(12, 'AW', 'ARUBA', 'Aruba', 'ABW', 533, 297),
(13, 'AU', 'AUSTRALIA', 'Australia', 'AUS', 36, 61),
(14, 'AT', 'AUSTRIA', 'Austria', 'AUT', 40, 43),
(15, 'AZ', 'AZERBAIJAN', 'Azerbaijan', 'AZE', 31, 994),
(16, 'BS', 'BAHAMAS', 'Bahamas', 'BHS', 44, 1242),
(17, 'BH', 'BAHRAIN', 'Bahrain', 'BHR', 48, 973),
(18, 'BD', 'BANGLADESH', 'Bangladesh', 'BGD', 50, 880),
(19, 'BB', 'BARBADOS', 'Barbados', 'BRB', 52, 1246),
(20, 'BY', 'BELARUS', 'Belarus', 'BLR', 112, 375),
(21, 'BE', 'BELGIUM', 'Belgium', 'BEL', 56, 32),
(22, 'BZ', 'BELIZE', 'Belize', 'BLZ', 84, 501),
(23, 'BJ', 'BENIN', 'Benin', 'BEN', 204, 229),
(24, 'BM', 'BERMUDA', 'Bermuda', 'BMU', 60, 1441),
(25, 'BT', 'BHUTAN', 'Bhutan', 'BTN', 64, 975),
(26, 'BO', 'BOLIVIA', 'Bolivia', 'BOL', 68, 591),
(27, 'BA', 'BOSNIA AND HERZEGOVINA', 'Bosnia and Herzegovina', 'BIH', 70, 387),
(28, 'BW', 'BOTSWANA', 'Botswana', 'BWA', 72, 267),
(29, 'BV', 'BOUVET ISLAND', 'Bouvet Island', NULL, NULL, 0),
(30, 'BR', 'BRAZIL', 'Brazil', 'BRA', 76, 55),
(31, 'IO', 'BRITISH INDIAN OCEAN TERRITORY', 'British Indian Ocean Territory', NULL, NULL, 246),
(32, 'BN', 'BRUNEI DARUSSALAM', 'Brunei Darussalam', 'BRN', 96, 673),
(33, 'BG', 'BULGARIA', 'Bulgaria', 'BGR', 100, 359),
(34, 'BF', 'BURKINA FASO', 'Burkina Faso', 'BFA', 854, 226),
(35, 'BI', 'BURUNDI', 'Burundi', 'BDI', 108, 257),
(36, 'KH', 'CAMBODIA', 'Cambodia', 'KHM', 116, 855),
(37, 'CM', 'CAMEROON', 'Cameroon', 'CMR', 120, 237),
(38, 'CA', 'CANADA', 'Canada', 'CAN', 124, 1),
(39, 'CV', 'CAPE VERDE', 'Cape Verde', 'CPV', 132, 238),
(40, 'KY', 'CAYMAN ISLANDS', 'Cayman Islands', 'CYM', 136, 1345),
(41, 'CF', 'CENTRAL AFRICAN REPUBLIC', 'Central African Republic', 'CAF', 140, 236),
(42, 'TD', 'CHAD', 'Chad', 'TCD', 148, 235),
(43, 'CL', 'CHILE', 'Chile', 'CHL', 152, 56),
(44, 'CN', 'CHINA', 'China', 'CHN', 156, 86),
(45, 'CX', 'CHRISTMAS ISLAND', 'Christmas Island', NULL, NULL, 61),
(46, 'CC', 'COCOS (KEELING) ISLANDS', 'Cocos (Keeling) Islands', NULL, NULL, 672),
(47, 'CO', 'COLOMBIA', 'Colombia', 'COL', 170, 57),
(48, 'KM', 'COMOROS', 'Comoros', 'COM', 174, 269),
(49, 'CG', 'CONGO', 'Congo', 'COG', 178, 242),
(50, 'CD', 'CONGO, THE DEMOCRATIC REPUBLIC OF THE', 'Congo, the Democratic Republic of the', 'COD', 180, 242),
(51, 'CK', 'COOK ISLANDS', 'Cook Islands', 'COK', 184, 682),
(52, 'CR', 'COSTA RICA', 'Costa Rica', 'CRI', 188, 506),
(53, 'CI', "COTE D''IVOIRE", "Cote D''Ivoire", 'CIV', 384, 225),
(54, 'HR', 'CROATIA', 'Croatia', 'HRV', 191, 385),
(55, 'CU', 'CUBA', 'Cuba', 'CUB', 192, 53),
(56, 'CY', 'CYPRUS', 'Cyprus', 'CYP', 196, 357),
(57, 'CZ', 'CZECH REPUBLIC', 'Czech Republic', 'CZE', 203, 420),
(58, 'DK', 'DENMARK', 'Denmark', 'DNK', 208, 45),
(59, 'DJ', 'DJIBOUTI', 'Djibouti', 'DJI', 262, 253),
(60, 'DM', 'DOMINICA', 'Dominica', 'DMA', 212, 1767),
(61, 'DO', 'DOMINICAN REPUBLIC', 'Dominican Republic', 'DOM', 214, 1809),
(62, 'EC', 'ECUADOR', 'Ecuador', 'ECU', 218, 593),
(63, 'EG', 'EGYPT', 'Egypt', 'EGY', 818, 20),
(64, 'SV', 'EL SALVADOR', 'El Salvador', 'SLV', 222, 503),
(65, 'GQ', 'EQUATORIAL GUINEA', 'Equatorial Guinea', 'GNQ', 226, 240),
(66, 'ER', 'ERITREA', 'Eritrea', 'ERI', 232, 291),
(67, 'EE', 'ESTONIA', 'Estonia', 'EST', 233, 372),
(68, 'ET', 'ETHIOPIA', 'Ethiopia', 'ETH', 231, 251),
(69, 'FK', 'FALKLAND ISLANDS (MALVINAS)', 'Falkland Islands (Malvinas)', 'FLK', 238, 500),
(70, 'FO', 'FAROE ISLANDS', 'Faroe Islands', 'FRO', 234, 298),
(71, 'FJ', 'FIJI', 'Fiji', 'FJI', 242, 679),
(72, 'FI', 'FINLAND', 'Finland', 'FIN', 246, 358),
(73, 'FR', 'FRANCE', 'France', 'FRA', 250, 33),
(74, 'GF', 'FRENCH GUIANA', 'French Guiana', 'GUF', 254, 594),
(75, 'PF', 'FRENCH POLYNESIA', 'French Polynesia', 'PYF', 258, 689),
(76, 'TF', 'FRENCH SOUTHERN TERRITORIES', 'French Southern Territories', NULL, NULL, 0),
(77, 'GA', 'GABON', 'Gabon', 'GAB', 266, 241),
(78, 'GM', 'GAMBIA', 'Gambia', 'GMB', 270, 220),
(79, 'GE', 'GEORGIA', 'Georgia', 'GEO', 268, 995),
(80, 'DE', 'GERMANY', 'Germany', 'DEU', 276, 49),
(81, 'GH', 'GHANA', 'Ghana', 'GHA', 288, 233),
(82, 'GI', 'GIBRALTAR', 'Gibraltar', 'GIB', 292, 350),
(83, 'GR', 'GREECE', 'Greece', 'GRC', 300, 30),
(84, 'GL', 'GREENLAND', 'Greenland', 'GRL', 304, 299),
(85, 'GD', 'GRENADA', 'Grenada', 'GRD', 308, 1473),
(86, 'GP', 'GUADELOUPE', 'Guadeloupe', 'GLP', 312, 590),
(87, 'GU', 'GUAM', 'Guam', 'GUM', 316, 1671),
(88, 'GT', 'GUATEMALA', 'Guatemala', 'GTM', 320, 502),
(89, 'GN', 'GUINEA', 'Guinea', 'GIN', 324, 224),
(90, 'GW', 'GUINEA-BISSAU', 'Guinea-Bissau', 'GNB', 624, 245),
(91, 'GY', 'GUYANA', 'Guyana', 'GUY', 328, 592),
(92, 'HT', 'HAITI', 'Haiti', 'HTI', 332, 509),
(93, 'HM', 'HEARD ISLAND AND MCDONALD ISLANDS', 'Heard Island and Mcdonald Islands', NULL, NULL, 0),
(94, 'VA', 'HOLY SEE (VATICAN CITY STATE)', 'Holy See (Vatican City State)', 'VAT', 336, 39),
(95, 'HN', 'HONDURAS', 'Honduras', 'HND', 340, 504),
(96, 'HK', 'HONG KONG', 'Hong Kong', 'HKG', 344, 852),
(97, 'HU', 'HUNGARY', 'Hungary', 'HUN', 348, 36),
(98, 'IS', 'ICELAND', 'Iceland', 'ISL', 352, 354),
(99, 'IN', 'INDIA', 'India', 'IND', 356, 91),
(100, 'ID', 'INDONESIA', 'Indonesia', 'IDN', 360, 62),
(101, 'IR', 'IRAN, ISLAMIC REPUBLIC OF', 'Iran, Islamic Republic of', 'IRN', 364, 98),
(102, 'IQ', 'IRAQ', 'Iraq', 'IRQ', 368, 964),
(103, 'IE', 'IRELAND', 'Ireland', 'IRL', 372, 353),
(104, 'IL', 'ISRAEL', 'Israel', 'ISR', 376, 972),
(105, 'IT', 'ITALY', 'Italy', 'ITA', 380, 39),
(106, 'JM', 'JAMAICA', 'Jamaica', 'JAM', 388, 1876),
(107, 'JP', 'JAPAN', 'Japan', 'JPN', 392, 81),
(108, 'JO', 'JORDAN', 'Jordan', 'JOR', 400, 962),
(109, 'KZ', 'KAZAKHSTAN', 'Kazakhstan', 'KAZ', 398, 7),
(110, 'KE', 'KENYA', 'Kenya', 'KEN', 404, 254),
(111, 'KI', 'KIRIBATI', 'Kiribati', 'KIR', 296, 686),
(112, 'KP', "KOREA, DEMOCRATIC PEOPLE''S REPUBLIC OF", "Korea, Democratic People''s Republic of", 'PRK', 408, 850),
(113, 'KR', 'KOREA, REPUBLIC OF', 'Korea, Republic of', 'KOR', 410, 82),
(114, 'KW', 'KUWAIT', 'Kuwait', 'KWT', 414, 965),
(115, 'KG', 'KYRGYZSTAN', 'Kyrgyzstan', 'KGZ', 417, 996),
(116, 'LA', "LAO PEOPLE''S DEMOCRATIC REPUBLIC", "Lao People''s Democratic Republic", 'LAO', 418, 856),
(117, 'LV', 'LATVIA', 'Latvia', 'LVA', 428, 371),
(118, 'LB', 'LEBANON', 'Lebanon', 'LBN', 422, 961),
(119, 'LS', 'LESOTHO', 'Lesotho', 'LSO', 426, 266),
(120, 'LR', 'LIBERIA', 'Liberia', 'LBR', 430, 231),
(121, 'LY', 'LIBYAN ARAB JAMAHIRIYA', 'Libyan Arab Jamahiriya', 'LBY', 434, 218),
(122, 'LI', 'LIECHTENSTEIN', 'Liechtenstein', 'LIE', 438, 423),
(123, 'LT', 'LITHUANIA', 'Lithuania', 'LTU', 440, 370),
(124, 'LU', 'LUXEMBOURG', 'Luxembourg', 'LUX', 442, 352),
(125, 'MO', 'MACAO', 'Macao', 'MAC', 446, 853),
(126, 'MK', 'MACEDONIA, THE FORMER YUGOSLAV REPUBLIC OF', 'Macedonia, the Former Yugoslav Republic of', 'MKD', 807, 389),
(127, 'MG', 'MADAGASCAR', 'Madagascar', 'MDG', 450, 261),
(128, 'MW', 'MALAWI', 'Malawi', 'MWI', 454, 265),
(129, 'MY', 'MALAYSIA', 'Malaysia', 'MYS', 458, 60),
(130, 'MV', 'MALDIVES', 'Maldives', 'MDV', 462, 960),
(131, 'ML', 'MALI', 'Mali', 'MLI', 466, 223),
(132, 'MT', 'MALTA', 'Malta', 'MLT', 470, 356),
(133, 'MH', 'MARSHALL ISLANDS', 'Marshall Islands', 'MHL', 584, 692),
(134, 'MQ', 'MARTINIQUE', 'Martinique', 'MTQ', 474, 596),
(135, 'MR', 'MAURITANIA', 'Mauritania', 'MRT', 478, 222),
(136, 'MU', 'MAURITIUS', 'Mauritius', 'MUS', 480, 230),
(137, 'YT', 'MAYOTTE', 'Mayotte', NULL, NULL, 269),
(138, 'MX', 'MEXICO', 'Mexico', 'MEX', 484, 52),
(139, 'FM', 'MICRONESIA, FEDERATED STATES OF', 'Micronesia, Federated States of', 'FSM', 583, 691),
(140, 'MD', 'MOLDOVA, REPUBLIC OF', 'Moldova, Republic of', 'MDA', 498, 373),
(141, 'MC', 'MONACO', 'Monaco', 'MCO', 492, 377),
(142, 'MN', 'MONGOLIA', 'Mongolia', 'MNG', 496, 976),
(143, 'MS', 'MONTSERRAT', 'Montserrat', 'MSR', 500, 1664),
(144, 'MA', 'MOROCCO', 'Morocco', 'MAR', 504, 212),
(145, 'MZ', 'MOZAMBIQUE', 'Mozambique', 'MOZ', 508, 258),
(146, 'MM', 'MYANMAR', 'Myanmar', 'MMR', 104, 95),
(147, 'NA', 'NAMIBIA', 'Namibia', 'NAM', 516, 264),
(148, 'NR', 'NAURU', 'Nauru', 'NRU', 520, 674),
(149, 'NP', 'NEPAL', 'Nepal', 'NPL', 524, 977),
(150, 'NL', 'NETHERLANDS', 'Netherlands', 'NLD', 528, 31),
(151, 'AN', 'NETHERLANDS ANTILLES', 'Netherlands Antilles', 'ANT', 530, 599),
(152, 'NC', 'NEW CALEDONIA', 'New Caledonia', 'NCL', 540, 687),
(153, 'NZ', 'NEW ZEALAND', 'New Zealand', 'NZL', 554, 64),
(154, 'NI', 'NICARAGUA', 'Nicaragua', 'NIC', 558, 505),
(155, 'NE', 'NIGER', 'Niger', 'NER', 562, 227),
(156, 'NG', 'NIGERIA', 'Nigeria', 'NGA', 566, 234),
(157, 'NU', 'NIUE', 'Niue', 'NIU', 570, 683),
(158, 'NF', 'NORFOLK ISLAND', 'Norfolk Island', 'NFK', 574, 672),
(159, 'MP', 'NORTHERN MARIANA ISLANDS', 'Northern Mariana Islands', 'MNP', 580, 1670),
(160, 'NO', 'NORWAY', 'Norway', 'NOR', 578, 47),
(161, 'OM', 'OMAN', 'Oman', 'OMN', 512, 968),
(162, 'PK', 'PAKISTAN', 'Pakistan', 'PAK', 586, 92),
(163, 'PW', 'PALAU', 'Palau', 'PLW', 585, 680),
(164, 'PS', 'PALESTINIAN TERRITORY, OCCUPIED', 'Palestinian Territory, Occupied', NULL, NULL, 970),
(165, 'PA', 'PANAMA', 'Panama', 'PAN', 591, 507),
(166, 'PG', 'PAPUA NEW GUINEA', 'Papua New Guinea', 'PNG', 598, 675),
(167, 'PY', 'PARAGUAY', 'Paraguay', 'PRY', 600, 595),
(168, 'PE', 'PERU', 'Peru', 'PER', 604, 51),
(169, 'PH', 'PHILIPPINES', 'Philippines', 'PHL', 608, 63),
(170, 'PN', 'PITCAIRN', 'Pitcairn', 'PCN', 612, 0),
(171, 'PL', 'POLAND', 'Poland', 'POL', 616, 48),
(172, 'PT', 'PORTUGAL', 'Portugal', 'PRT', 620, 351),
(173, 'PR', 'PUERTO RICO', 'Puerto Rico', 'PRI', 630, 1787),
(174, 'QA', 'QATAR', 'Qatar', 'QAT', 634, 974),
(175, 'RE', 'REUNION', 'Reunion', 'REU', 638, 262),
(176, 'RO', 'ROMANIA', 'Romania', 'ROM', 642, 40),
(177, 'RU', 'RUSSIAN FEDERATION', 'Russian Federation', 'RUS', 643, 70),
(178, 'RW', 'RWANDA', 'Rwanda', 'RWA', 646, 250),
(179, 'SH', 'SAINT HELENA', 'Saint Helena', 'SHN', 654, 290),
(180, 'KN', 'SAINT KITTS AND NEVIS', 'Saint Kitts and Nevis', 'KNA', 659, 1869),
(181, 'LC', 'SAINT LUCIA', 'Saint Lucia', 'LCA', 662, 1758),
(182, 'PM', 'SAINT PIERRE AND MIQUELON', 'Saint Pierre and Miquelon', 'SPM', 666, 508),
(183, 'VC', 'SAINT VINCENT AND THE GRENADINES', 'Saint Vincent and the Grenadines', 'VCT', 670, 1784),
(184, 'WS', 'SAMOA', 'Samoa', 'WSM', 882, 684),
(185, 'SM', 'SAN MARINO', 'San Marino', 'SMR', 674, 378),
(186, 'ST', 'SAO TOME AND PRINCIPE', 'Sao Tome and Principe', 'STP', 678, 239),
(187, 'SA', 'SAUDI ARABIA', 'Saudi Arabia', 'SAU', 682, 966),
(188, 'SN', 'SENEGAL', 'Senegal', 'SEN', 686, 221),
(189, 'CS', 'SERBIA AND MONTENEGRO', 'Serbia and Montenegro', NULL, NULL, 381),
(190, 'SC', 'SEYCHELLES', 'Seychelles', 'SYC', 690, 248),
(191, 'SL', 'SIERRA LEONE', 'Sierra Leone', 'SLE', 694, 232),
(192, 'SG', 'SINGAPORE', 'Singapore', 'SGP', 702, 65),
(193, 'SK', 'SLOVAKIA', 'Slovakia', 'SVK', 703, 421),
(194, 'SI', 'SLOVENIA', 'Slovenia', 'SVN', 705, 386),
(195, 'SB', 'SOLOMON ISLANDS', 'Solomon Islands', 'SLB', 90, 677),
(196, 'SO', 'SOMALIA', 'Somalia', 'SOM', 706, 252),
(197, 'ZA', 'SOUTH AFRICA', 'South Africa', 'ZAF', 710, 27),
(198, 'GS', 'SOUTH GEORGIA AND THE SOUTH SANDWICH ISLANDS', 'South Georgia and the South Sandwich Islands', NULL, NULL, 0),
(199, 'ES', 'SPAIN', 'Spain', 'ESP', 724, 34),
(200, 'LK', 'SRI LANKA', 'Sri Lanka', 'LKA', 144, 94),
(201, 'SD', 'SUDAN', 'Sudan', 'SDN', 736, 249),
(202, 'SR', 'SURINAME', 'Suriname', 'SUR', 740, 597),
(203, 'SJ', 'SVALBARD AND JAN MAYEN', 'Svalbard and Jan Mayen', 'SJM', 744, 47),
(204, 'SZ', 'SWAZILAND', 'Swaziland', 'SWZ', 748, 268),
(205, 'SE', 'SWEDEN', 'Sweden', 'SWE', 752, 46),
(206, 'CH', 'SWITZERLAND', 'Switzerland', 'CHE', 756, 41),
(207, 'SY', 'SYRIAN ARAB REPUBLIC', 'Syrian Arab Republic', 'SYR', 760, 963),
(208, 'TW', 'TAIWAN, PROVINCE OF CHINA', 'Taiwan, Province of China', 'TWN', 158, 886),
(209, 'TJ', 'TAJIKISTAN', 'Tajikistan', 'TJK', 762, 992),
(210, 'TZ', 'TANZANIA, UNITED REPUBLIC OF', 'Tanzania, United Republic of', 'TZA', 834, 255),
(211, 'TH', 'THAILAND', 'Thailand', 'THA', 764, 66),
(212, 'TL', 'TIMOR-LESTE', 'Timor-Leste', NULL, NULL, 670),
(213, 'TG', 'TOGO', 'Togo', 'TGO', 768, 228),
(214, 'TK', 'TOKELAU', 'Tokelau', 'TKL', 772, 690),
(215, 'TO', 'TONGA', 'Tonga', 'TON', 776, 676),
(216, 'TT', 'TRINIDAD AND TOBAGO', 'Trinidad and Tobago', 'TTO', 780, 1868),
(217, 'TN', 'TUNISIA', 'Tunisia', 'TUN', 788, 216),
(218, 'TR', 'TURKEY', 'Turkey', 'TUR', 792, 90),
(219, 'TM', 'TURKMENISTAN', 'Turkmenistan', 'TKM', 795, 7370),
(220, 'TC', 'TURKS AND CAICOS ISLANDS', 'Turks and Caicos Islands', 'TCA', 796, 1649),
(221, 'TV', 'TUVALU', 'Tuvalu', 'TUV', 798, 688),
(222, 'UG', 'UGANDA', 'Uganda', 'UGA', 800, 256),
(223, 'UA', 'UKRAINE', 'Ukraine', 'UKR', 804, 380),
(224, 'AE', 'UNITED ARAB EMIRATES', 'United Arab Emirates', 'ARE', 784, 971),
(225, 'GB', 'UNITED KINGDOM', 'United Kingdom', 'GBR', 826, 44),
(226, 'US', 'UNITED STATES', 'United States', 'USA', 840, 1),
(227, 'UM', 'UNITED STATES MINOR OUTLYING ISLANDS', 'United States Minor Outlying Islands', NULL, NULL, 1),
(228, 'UY', 'URUGUAY', 'Uruguay', 'URY', 858, 598),
(229, 'UZ', 'UZBEKISTAN', 'Uzbekistan', 'UZB', 860, 998),
(230, 'VU', 'VANUATU', 'Vanuatu', 'VUT', 548, 678),
(231, 'VE', 'VENEZUELA', 'Venezuela', 'VEN', 862, 58),
(232, 'VN', 'VIET NAM', 'Viet Nam', 'VNM', 704, 84),
(233, 'VG', 'VIRGIN ISLANDS, BRITISH', 'Virgin Islands, British', 'VGB', 92, 1284),
(234, 'VI', 'VIRGIN ISLANDS, U.S.', 'Virgin Islands, U.s.', 'VIR', 850, 1340),
(235, 'WF', 'WALLIS AND FUTUNA', 'Wallis and Futuna', 'WLF', 876, 681),
(236, 'EH', 'WESTERN SAHARA', 'Western Sahara', 'ESH', 732, 212),
(237, 'YE', 'YEMEN', 'Yemen', 'YEM', 887, 967),
(238, 'ZM', 'ZAMBIA', 'Zambia', 'ZMB', 894, 260),
(239, 'ZW', 'ZIMBABWE', 'Zimbabwe', 'ZWE', 716, 263);
```

And in Bigquery run this sql query. Now we have the table `country`.

The next step is to define the models that will generate the fact table and dimension tables.

First we create a new folder `transform` in the dbt.

And then create a `dim_customer.sql` in `transform`

```sql
-- dim_customer.sql

-- Create the dimension table
WITH customer_cte AS (
	SELECT DISTINCT
	    {{ dbt_utils.generate_surrogate_key(['CustomerID', 'Country']) }} as customer_id,
	    Country AS country
	FROM {{ source('retail', 'raw_invoices') }}
	WHERE CustomerID IS NOT NULL
)
SELECT
    t.*,
	cm.iso
FROM customer_cte t
LEFT JOIN {{ source('retail', 'country') }} cm ON t.country = cm.nicename
```

And a dim_datetime.sql in transform

```sql
-- dim_datetime.sql

-- Create a CTE to extract date and time components
WITH datetime_cte AS (  
  SELECT DISTINCT
    InvoiceDate AS datetime_id,
    CASE
      WHEN LENGTH(InvoiceDate) = 16 THEN
        -- Date format: "DD/MM/YYYY HH:MM"
        PARSE_DATETIME('%m/%d/%Y %H:%M', InvoiceDate)
      WHEN LENGTH(InvoiceDate) <= 14 THEN
        -- Date format: "MM/DD/YY HH:MM"
        PARSE_DATETIME('%m/%d/%y %H:%M', InvoiceDate)
      ELSE
        NULL
    END AS date_part,
  FROM {{ source('retail', 'raw_invoices') }}
  WHERE InvoiceDate IS NOT NULL
)
SELECT
  datetime_id,
  date_part as datetime,
  EXTRACT(YEAR FROM date_part) AS year,
  EXTRACT(MONTH FROM date_part) AS month,
  EXTRACT(DAY FROM date_part) AS day,
  EXTRACT(HOUR FROM date_part) AS hour,
  EXTRACT(MINUTE FROM date_part) AS minute,
  EXTRACT(DAYOFWEEK FROM date_part) AS weekday
FROM datetime_cte
```

and dim_product.sql in transform

```sql
-- dim_product.sql
-- StockCode isn't unique, a product with the same id can have different and prices
-- Create the dimension table
SELECT DISTINCT
    {{ dbt_utils.generate_surrogate_key(['StockCode', 'Description', 'UnitPrice']) }} as product_id,
		StockCode AS stock_code,
    Description AS description,
    UnitPrice AS price
FROM {{ source('retail', 'raw_invoices') }}
WHERE StockCode IS NOT NULL
AND UnitPrice > 0
```

and the fact table.

fct_invoices.sql  in transform

```sql
-- fct_invoices.sql

-- Create the fact table by joining the relevant keys from dimension table
WITH fct_invoices_cte AS (
    SELECT
        InvoiceNo AS invoice_id,
        InvoiceDate AS datetime_id,
        {{ dbt_utils.generate_surrogate_key(['StockCode', 'Description', 'UnitPrice']) }} as product_id,
        {{ dbt_utils.generate_surrogate_key(['CustomerID', 'Country']) }} as customer_id,
        Quantity AS quantity,
        Quantity * UnitPrice AS total
    FROM {{ source('retail', 'raw_invoices') }}
    WHERE Quantity > 0
)
SELECT
    invoice_id,
    dt.datetime_id,
    dp.product_id,
    dc.customer_id,
    quantity,
    total
FROM fct_invoices_cte fi
INNER JOIN {{ ref('dim_datetime') }} dt ON fi.datetime_id = dt.datetime_id
INNER JOIN {{ ref('dim_product') }} dp ON fi.product_id = dp.product_id
INNER JOIN {{ ref('dim_customer') }} dc ON fi.customer_id = dc.customer_id
```

Run the models

move the folder transform into models

```bash
astro dev bash
source /usr/local/airflow/dbt_venv/bin/activate
cd include/dbt 
dbt deps
dbt run --profiles-dir /usr/local/airflow/include/dbt/
```



Create a new file `cosmos_config.py`in the dbt.

```python
# include/dbt/cosmos_config.py

from cosmos.config import ProfileConfig, ProjectConfig
from pathlib import Path

DBT_CONFIG = ProfileConfig(
    profile_name='retail',
    target_name='dev',
    profiles_yml_filepath=Path('/usr/local/airflow/include/dbt/profiles.yml')
)

DBT_PROJECT_CONFIG = ProjectConfig(
    dbt_project_path='/usr/local/airflow/include/dbt/',
)
```

And we go back to `retail.py`

```python
from include.dbt.cosmos_config import DBT_PROJECT_CONFIG, DBT_CONFIG
from cosmos.airflow.task_group import DbtTaskGroup
from cosmos.constants import LoadMode
from cosmos.config import ProjectConfig, RenderConfig

transform = DbtTaskGroup(
        group_id='transform',
        project_config=DBT_PROJECT_CONFIG,
        profile_config=DBT_CONFIG,
        render_config=RenderConfig(
            load_method=LoadMode.DBT_LS,
            select=['path:models/transform']
        )
    )
```

This is some unexcepted issue :

so we add 2 lines

```python
# dbt_project.yml

name: 'retail'
config-version: 2 #添加这一行
version: '1.0'  # 添加这一行


profile: 'retail'

models:
  retail:
    materialized: table
```

And we need to use soda to check the data is still correct.

we create a new folder `transform` in `sources`.

And create a new yml `dim_customer.yml` in transform

```yaml
# dim_customer.yml
checks for dim_customer:
  - schema:
      fail:
        when required column missing: 
          [customer_id, country]
        when wrong column type:
          customer_id: string
          country: string
  - duplicate_count(customer_id) = 0:
      name: All customers are unique
  - missing_count(customer_id) = 0:
      name: All customers have a key
```



and create a `dim_datetime.yml` in transform

```yaml
# dim_datetime.yml
checks for dim_datetime:
  # Check fails when product_key or english_product_name is missing, OR
  # when the data type of those columns is other than specified
  - schema:
      fail:
        when required column missing: [datetime_id, datetime]
        when wrong column type:
          datetime_id: string
          datetime: datetime
  # Check failes when weekday is not in range 0-6
  - invalid_count(weekday) = 0:
      name: All weekdays are in range 0-6
      valid min: 0
      valid max: 6
  # Check fails when customer_id is not unique
  - duplicate_count(datetime_id) = 0:
      name: All datetimes are unique
  # Check fails when any NULL values exist in the column
  - missing_count(datetime_id) = 0:
      name: All datetimes have a key
```

and same for dim_product.yml

```yaml
# dim_product.yml
checks for dim_product:
  # Check fails when product_key or english_product_name is missing, OR
  # when the data type of those columns is other than specified
  - schema:
      fail:
        when required column missing: [product_id, description, price]
        when wrong column type:
          product_id: string
          description: string
          price: float64
  # Check fails when customer_id is not unique
  - duplicate_count(product_id) = 0:
      name: All products are unique
  # Check fails when any NULL values exist in the column
  - missing_count(product_id) = 0:
      name: All products have a key
  # Check fails when any prices are negative
  - min(price):
      fail: when < 0
```

and fct_invoices.yml

```yaml
# fct_invoices.yml
checks for fct_invoices:
  # Check fails when invoice_id, quantity, total is missing or
  # when the data type of those columns is other than specified
  - schema:
      fail:
        when required column missing: 
          [invoice_id, product_id, customer_id, datetime_id, quantity, total]
        when wrong column type:
          invoice_id: string
          product_id: string
          customer_id: string
          datetime_id: string
          quantity: int
          total: float64
  # Check fails when NULL values in the column
  - missing_count(invoice_id) = 0:
      name: All invoices have a key
  # Check fails when the total of any invoices is negative
  - failed rows:
      name: All invoices have a positive total amount
      fail query: |
        SELECT invoice_id, total
        FROM fct_invoices
        WHERE total < 0
```

And then add a new task

```python
@task.external_python(python='/usr/local/airflow/soda_venv/bin/python')
    def check_transform(scan_name='check_transform', checks_subpath='transform'):
        from include.soda.checks.sources.check_function import check

        return check(scan_name, checks_subpath)
```

and then tesk 

```bash
astro dev bash
airflow tasks test retail check_transform 2024-08-15
```

The last step of this data by plan is to create three tables that will correspond to metric derived from the dimension and the fact tables by using dbt.

So create a new folder `report` in dbt/models 

and a file `report_customer_invoices.sql` in `report`

```sql
-- report_customer_invoices.sql
SELECT
  c.country,
  c.iso,
  COUNT(fi.invoice_id) AS total_invoices,
  SUM(fi.total) AS total_revenue
FROM {{ ref('fct_invoices') }} fi
JOIN {{ ref('dim_customer') }} c ON fi.customer_id = c.customer_id
GROUP BY c.country, c.iso
ORDER BY total_revenue DESC
LIMIT 10
```

and a file `report_product_invoices.sql `  in `report`

```sql
-- report_product_invoices.sql
SELECT
  p.product_id,
  p.stock_code,
  p.description,
  SUM(fi.quantity) AS total_quantity_sold
FROM {{ ref('fct_invoices') }} fi
JOIN {{ ref('dim_product') }} p ON fi.product_id = p.product_id
GROUP BY p.product_id, p.stock_code, p.description
ORDER BY total_quantity_sold DESC
LIMIT 10
```

and report_year_invoices.sql in `report`

```sql
-- report_year_invoices.sql
SELECT
  dt.year,
  dt.month,
  COUNT(DISTINCT fi.invoice_id) AS num_invoices,
  SUM(fi.total) AS total_revenue
FROM {{ ref('fct_invoices') }} fi
JOIN {{ ref('dim_datetime') }} dt ON fi.datetime_id = dt.datetime_id
GROUP BY dt.year, dt.month
ORDER BY dt.year, dt.month
```

And we add a new task in retail.py

```python
report = DbtTaskGroup(
        group_id='report',
        project_config=DBT_PROJECT_CONFIG,
        profile_config=DBT_CONFIG,
        render_config=RenderConfig(
            load_method=LoadMode.DBT_LS,
            select=['path:models/report']
        )
    )
```



And we want to make sure that those metrics  are correct before reflecting them on the bashboard.

so we create  a folder `report` in soda/checks

and a file `report_customer_invoices.yml` in report

```yaml
# report_customer_invoices.yml
checks for report_customer_invoices:
  # Check fails if the country column has any missing values
  - missing_count(country) = 0:
      name: All customers have a country
  # Check fails if the total invoices is lower or equal to 0
  - min(total_invoices):
      fail: when <= 0
```

and report_product_invoices.yml in report

```yaml
# report_product_invoices.yml
checks for report_product_invoices:
  # Check fails if the stock_code column has any missing values
  - missing_count(stock_code) = 0:
      name: All products have a stock code
  # Check fails if the total quanity sold is lower or equal to 0
  - min(total_quantity_sold):
      fail: when <= 0
```

and report_year_invoices.yml in report

```yaml
# report_year_invoices.yml
checks for report_year_invoices:
  # Check fails if the number of invoices for a year is negative
  - min(num_invoices):
      fail: when < 0
```

And add the task

```python
from airflow.models.baseoperator import chain
# remember to remove the check_load()and check_transform() before.
    @task.external_python(python='/usr/local/airflow/soda_venv/bin/python')
    def check_report(scan_name='check_report', checks_subpath='report'):
        from include.soda.checks.sources.check_function import check

        return check(scan_name, checks_subpath)

    chain(
        upload_csv_to_gcs,
        create_retail_dataset,
        gcs_to_raw,
        check_load(),
        transform,
        check_transform(),
        report,
        check_report(),
    )
```

🥇 Well done! we’ve finished the data pipeline

### Create the dashboard 

First create a `docker-compose.override.yml` at the  root  of the project.

```yml
version: "3.1"
services:
  metabase:
    image: metabase/metabase:v0.46.6.4
    volumes:
      - ./include/metabase-data:/metabase-data
    environment:
      - MB_DB_FILE=/metabase-data/metabase.db
    ports:
      - 3000:3000
    restart: always
```



and 

```bash
astro dev restart
```

and go to localhost:3000

and 

```bash
astro dev bash
source dbt_venv/bin/activate
cd include/dbt/
dbt deps
dbt run --select path:models/report --profiles-dir /usr/local/airflow/include/dbt/
```



Then we go to localhost:3000 , setting-> admin settings->databases->click alan->click sync, re-scan. and save 

click exit admin  -> New->Question