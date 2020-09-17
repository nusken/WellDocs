- [Welligence Documents](#welligence-documents)

  - [Setup](#setup)
    - [Native](#setup)
    - [Docker](#docker)
    - [Machine learning](#machine-learning)
  - [Shard config](#shard-config)
  - [Workflow](#workflow)
  - [Coding](#coding)
    - [Import Job](#import-job)
    - [Crawl/Fetch Job](#crawlfetch-job)
    - [Valuation Page](#valuation-page)
    - [Map Page](#map-page)
    - [Run R](#run-r)
    - [Generate Excel Report](#generate-excel-report)
    - [Generate Enhanced Report PDF](#generate-enhanced-report-pdf)
    - [Cron Job](#cron-job)
  - [Server Config](#server-config)
    - [ML code](#ml-code)
    - [Backup Database](#backup-database)
  - [Jenkins](#jenkins)
    - [Run Job](#run-job)
    - [Config Job](#config-job)
  - [Worker Config](#server-config)
    - [Local config](#local-config)
    - [Jenkin config](#jenkin-config)
  - [Release Step](#release-step)
    - [Major version](#major-version)
    - [Minor version](#minor-version)
  - [Fix Template](#fix-template)
  - [Create New Country](#create-new-country)

# Welligence Documents

## Setup

sau khi clone code từ `https://github.com/welligence/WellStack` về thì bắt đầu cài đặt

### Native

Yêu cầu:

- đã cài rvm
- đã cài bundler
- đã cài postgresql
- ổ cứng còn trống hơn 30 GB
- đã config git

#### Install ruby

xem file `.ruby-version` để cài đặt ruby đúng version

sau khi cài đặt xong thì tiếp tục bundle install

```
bundle install
```

#### Setup Database

1. tạo 2 database là `welligence` và `welligence_brazil`
2. download 2 bản backup trên `staging` của shard `master` và `brazil`
3. giải nén và restore ra 2 database

#### Setup code

1. Copy database `database.yml.example`, rename to `database.yml` and put your postgresql configuration in to it.
2. Copy `config/application.yml.example`, rename to `application.yml`.
3. Copy `config/secrets.yml.example`, rename to `secrets.yml` and put `secret_key_base` dummy string.
4. Run `rails server` and double check

### Docker

Yêu cầu:

- đã cài đặt docker, docker-compose
- có kiến thức về docker, container
- ở môi trường development thì dùng file `Dockerfile.dev`, môi trường staging thì xử dụng `Dockerfile`, tương tự với `docker-compose-dev.yml` và `docker-compose.yml`

1. Create file `config/database.yml`

```
default: &default
  adapter: postgresql
  encoding: unicode
  pool: 1000
  username: <%= ENV.fetch("POSTGRES_USERNAME") %>
  password: <%= ENV.fetch("POSTGRES_PASSWORD") %>
  host: <%= ENV.fetch("POSTGRES_HOST") %>
  database: welligence

development:
  <<: *default

test:
  <<: *default
  database: welligence_test

```

2. Copy `config/application.yml.example`, rename to `application.yml`.
3. Copy `config/secrets.yml.example`, rename to `secrets.yml` and put `secret_key_base` dummy string.
4. Build docker image with your `AUTH_TOKEN` and `WELLML_BRANCH` ( should be master as default )

```
docker build -t welligence -f Dockerfile.dev --build-arg AUTH_TOKEN=0e458dd18f42dfb0683b6b67ebc61060d3820b33 --build-arg WELLML_BRANCH=master .
```

5. start docker-compose

download image and build docker containers

```
docker-compose -f docker-compose-dev.yml build

```

start docker containers

```
docker-compose -f docker-compose-dev.yml up
```

7. connect to postgres docker and create database ( example brazil )

chú ý port cho đúng

```
psql -U postgres -h localhost -p 5433

CREATE DATABASE welligence_brazil
```

in another terminal, extrart master database and brazil database to `sql.gz` file, then restore to docker database

```
gunzip /master_backup/databases/PostgreSQL.sql.gz | psql -U postgres -h localhost -p 5433 -d welligence
gunzip /brazil_backup/databases/PostgreSQL.sql.gz | psql -U postgres -h localhost -p 5433 -d welligence_brazil
```

8. connect to `localhost:3000`

9. Root folder is mount into docker container, so you don't have to restart docker-compose

#### Debug with docker

1. start docker-compose

```
docker-compose -f docker-compose-dev.yml up
```

2. attach container app

```
docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                            NAMES
58b188d031ed        welligence            "./dscripts/start_ra…"   11 minutes ago      Up 11 minutes       0.0.0.0:3000->3000/tcp, 8787/tcp                 wellstack_app_1
75e2eafd6900        welligence            "./dscripts/start_we…"   11 minutes ago      Up 11 minutes       0.0.0.0:3035->3035/tcp, 8787/tcp                 wellstack_webpacker_1
62aa0aac9960        postgres:12-alpine    "docker-entrypoint.s…"   11 minutes ago      Up 11 minutes       0.0.0.0:5433->5432/tcp                           wellstack_db_1
cf54af3afbee        redis:5-alpine        "docker-entrypoint.s…"   2 days ago          Up 11 minutes       6379/tcp                                         wellstack_redis_1
2e6fd2a3f455        elasticsearch:6.8.6   "/usr/local/bin/dock…"   2 days ago          Up 11 minutes       0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   wellstack_elasticsearch_1


docker attach wellstack_app_1
```

3. Debug like native

#### Run specific service

In case after start docker-compose and you want to start another service ( sidekiq, kibana ), you could

```
docker-compose up sidekiq
docker-compose up kibana
```

#### Run Command with App Service

```
docker exec -it wellstack_app_1 rails c # to debug
docker exec -it wellstack_app_1 rails db:migrate # to migrate database
```

#### Start sidekiq

1. Open `docker-compse-dev.yml` and uncomment service sidekiq
2. start docker-compose again or only start sidekiq

```
docker-compose -f docker-compose-dev.yml up sidekiq
```

#### Working with Kibana

1. Open `docker-compse-dev.yml` and uncomment service sidekiq
2. start docker-compose again or only start service kibana

```
docker-compose -f docker-compose-dev.yml up kibana
```

3. Connect `localhost:5601`

### Machine Learning

1. Tạo folder `machine_learning` và cd vào folder đó
2. clone code của các repo machine learning vào folder này với tên tương ứng với nước

```
git clone git@github.com:welligence/BrazilML.git brazil
git clone git@github.com:welligence/ArgMl.git argentina

```

3. Install R với version `3.4.3` cho giống với môi trường staging
4. xem file `install_package_r.sh` và làm theo để cài các package
5. Chạy thử 1 field xem có issue gì không

## Shard config

Welligence là 1 app multi sharding, gồm:

- 1 shard master, chứa các data chung ( users, user downloads, ...)
- Mỗi nước sẽ có 1 DB shard chứa data của riêng nước đó, vd DB `welligence_brazil` sẽ chứa thông tin chung & riêng Brazil mà thôi, tương tự với `welligence_mexico`, `welligence_colombia`, ...
- Mình có config 1 shard sample cho welligence_sample database, db này sẽ dùng để làm template khi create new country database

Dùng gem Octopus để handle việc này, shard config này được thực hiện trong file `shard_configuration.rb`

```ruby
return if Rails.env.test?

ENABLED_COUNTRY_DBS = [
  :angola,
  :argentina,
  :bahamas,
  ...
].freeze

global_conn_config = ActiveRecord::Base.connection_config

if ENV['COUNTRY_CODE']
  global_conn_config.merge!(database: "welligence_#{ENV['COUNTRY_CODE']}")
end

begin
  shards = {
    my_shards: {
      'master' => global_conn_config,
      'sample' => global_conn_config.merge(database: 'welligence_sample'),
    },
  }

  ENABLED_COUNTRY_DBS.each do |country_code|
    shards[:my_shards][country_code] =
      global_conn_config.merge(database: "welligence_#{country_code}")
  end

  Octopus.setup do |config|
    config.environments = [Rails.env]
    config.shards = shards[:my_shards]
  end
rescue ActiveRecord::StatementInvalid => e
  Raven.capture_exception(e)

  puts e
end
```

## Workflow

do welligence là 1 app với nhiều country, data đã chia ra thành các database riêng nên gitflow cũng sẽ được chia thành các nhảnh nhỏ
sẽ gồm các nhánh: master, staging, release, .... trong đó:

- master: là nhánh chứa code đã được test trên live
- staging: nhánh này là nhánh chứa code của tất cả các branch, dev khi muốn deploy code lên server staging đều merge vào nhánh này và deploy lên staging
- release/2.46_xxx: đây là các nhánh chứa code release của các nước, vd sau khi làm feature 123 và đã được client check trên staging rồi thì mình sẽ merge code vào nhánh release tương ứng ( vd làm cho nước brazil thì checkout và merge vào nhánh release/2.46_brazil)
- release/2.45.x: đây là nhánh chứa code release và chạy trên live, khi release mới thì đều checkout từ nhánh đang chạy trên live và tăng thêm 1 minor, đồng thời merge code của nhánh release country + update version.html

### khi làm feature bình thường

- đọc card và clear requirement trước khi làm, có thể hỏi dev khác, nếu vẫn ko rõ thì hỏi client
- khi bắt đầu làm thì move card từ `backlog` sang `doing`
- xem feature đó là cho nước nào và checkout từ nhánh release country của nước đó, vd card `https://trello.com/c/bUr7X1Xr/3719-med-brazil-new-enhanced-asset-report-slides` là feature cho nước Brazil thì nên checkout từ nhánh `release/2.46_bra`
- tạo nhánh có tên là `features/xxx_yyy` với `xxx` là số của card feature đó, `yyy` là mô tả tên nhánh đó, vd feature `features/3719_new_slide_brazil`
- với mỗi commit thì trong message commit phải có số card đó, vd làm card 3719 thì commit message là `3719: add new slide`
- sau khi đã hoàn thành chức năng ở local, cần deploy lên staging để client test, sẽ merge code của nhánh mình đang làm vào nhánh `staging`, sau đó dùng `jenkins` để deploy lên môi trường staging
- với job import thì sẽ ssh lên server staging, đồng thời up file csv/xlsx vào folder `/data_files/xxx` với xxx là số card, sau đó mở rails console lên và import file vừa up lên
- báo client check tính năng đó, đồng thời kéo card qua `Delivered in Staging`
- nếu họ có yêu cầu gì cần update trong card đó thì vẫn tiếp tục làm trên nhánh đó rồi deploy và báo check lại
- sau khi client đã check và confirm card này đã work thì client sẽ kéo qua `Checked in Staging`
- sau đó tiến hành tạo PR vào nhánh release của nước checkout trước đó, vd với card trên thì tạo PR vào nhánh `release/2.46_bra`, title PR là `3719: ABC` và comment là link card đó
- sau khi đã được review và merge vào nhánh release thì sẽ kéo qua phần `Merged into the release branch`, đồng thòi comment `merged into abc` với abc là nhánh release

### Khi làm card hotfix hoặc có cờ straight to live

quy trình như trên kèm theo

- những card này nên được checkout từ nhánh release mới nhất
- sau khi client đã check ổn hết rồi thì sẽ tạo PR vào nhánh release mới nhất + `1 minor` + update version html
- sau khi đã được review và merge rồi thì deploy nhánh release lên server live, đồng thời kéo card và request client check lại lần nữa

## Coding

### Import Job

đây là loại job thường thấy nhất trong app, vd client có 1 file CSV/EXCEL và yêu cầu import vào trong database

có 2 loại job with shard:

- nếu đã biết là dùng cho 1 country => ScraperJobWithShard
- nếu dùng cho nhiều country => ApplicationJobWithShard

#### các bước làm:

1. clear requirement với khách hàng, hỏi xem là cột nào sẽ import vào cột nào, thường sẽ có 2 loại cột là identity column và data column ( identity column là cột dùng để xác định/ tìm ra resource cần phải update/import, còn data column là cột chứa data để import vào database)

vd

```csv
well_name, well_depth
ABC, 123
XYZ, 456
```

trong file csv này có 2 cột là well_name và well_depth, trong đó well_name là identity column, dùng nó để tìm ra well cần update còn well_depth là data column chứa thông tin để import vào database

2. cần xác định job của nước nào và tạo job trong thư mục tương ứng, vd job của `Brazil` thì vào vào folder `app/jobs/brazil` để tạo job mới, lưu ý job mới tạo phải kế thừa từ class `ScraperJobWithShard` , đồng thòi có constant `COUNTRY_NAME`

```ruby
module Brazil
  class CreateDuplicateAssetJob < ScraperJobWithShard
    COUNTRY_NAME = 'Brazil'

    # your code here
```

3. tiếp theo sẽ implement hàm perform của job này, thường sẽ nhận 1 argument là file_path, đọc nội dung file cần import thành 1 array 2 chiều, sau đó loop qua từng row trong mãng 2 chiều trên và import vào database

4. đễ cho clean code, hầu hết sẽ không import trực tiếp trong job mà sẻ đẩy data vào 1 parser tương thích, vd

```ruby
rows.each do |row|
  Rows::Brazil::WellProductionDataParser.new(
    row,
    self,
    country,
  ).perform
end
```

5. trong parser sẽ tiếp tục implement, cần chú ý trong hàm `convert_row_to_attrs`, hàm này sẽ dọc file `attrs_mapping.yml` và các file yaml của các nước, vd: `brazil_attrs_mapping.yml`,`mexico_attrs_mapping.yml`, cần tạo `model_type` tương thích

vd với file csv ở trên thì

```yml
update_well_depth:
  main:
    - well_name
    - well_depth
```

hàm `convert_row_to_attrs` sẽ convert mãng 1 chiều row thành 1 hash data

```ruby
=> attrs
{
  well_name: 'ABC',
  well_depth: 123
}
```

sau khi đã có hash data này, sẽ lần lượt đọc thông tin để import, vd tìm well thông qua `attrs[:well_name]`, sau đó update well_depth vào well vừa tìm được

6. sau khi import xong thì check lại và merge vào nhánh `staging`, sau đó deploy lên staging
7. scp file lên staging, folder `data_files/xxx`, với xxx là số card, sau đó vào rails c và chạy lại job trên staging
8. check lại và báo khách hàng check

### Crawl/Fetch Job

job này tương tự như import job, nhưng thay vì khách hàng đưa file sẵn thì sẽ kêu crawling từ 1 nguồn nào đó ( khách hàng sẽ đưa link )

mình sẽ tìm cách để từ link đó tải file về, có thể là file excel, csv, pdf hoặc html... mục đích cuối cùng là để convert thành mãng 2 chiều giống job import, sau đó cũng đẩy vào parser để xử lí thông tin vừa scrawl được

### Valuation Page

- lưu ý, trước khi test cần generate elastichsearch data cho các nước

page này gồm 2 phần chính:

- filter section
- assets section

#### filter section

filter gồm 2 request chính:

- request `fetch_country_relative_data_overview` để lấy thông tin dropdown element
- request `assets` để call service để lấy thông tin html và show lên trên screen ( phần assets section )

1. mỗi khi select 1 element bất kì trong khung filter, đều sẽ call `fetch_country_relative_data_overview` để lấy thông tin dropdown mới phù hợp. vd lúc đầu ở mục asset có rất nhiều field/block, nhưng khi filter chọn country là brazil thì khi chọn lại asset chỉ còn lại 1 số field của brazil thôi
2. đồng thời sẽ call `assets` để lấy thông tin match với filter đang chọn và show lên, vd lúc đầu nếu ko chọn gì thì sẽ có 1000 assets, khi chọn country brazil thì sẽ chỉ còn show
   `Assets 200 Assets Found`
3. Filter bao gồm cả pagination và sortBy

#### Asset section

khi FE call `assets` thì BE sẽ query trong elastichsearch để tìm ra những asset phù hợp thông qua service `Customer::AssetOverview::SearchAssetsService`, service này sẽ trả về thông tin số asset phù hợp với điều kiện filter

Sau đó ta sẽ đẩy data này vào `Customer::AssetOverview::AssetFacade` để render html cho clean và chính xác, tầng view chỉ việc gọi những hàm đã implement trong class AssetFacade, còn việc implement như thế nào thì chi tiết sẽ vào trong hàm của class đó => tách biệt logic và view

Ngoài ra ta còn dùng cache Redis

```ruby
- facades.each do |facade|
  = generate_asset(facade)
```

```ruby
def generate_asset(facade)
  Rails.cache.fetch(facade.cache_key) { render partial: 'customer/assets/asset', locals: { facade: facade } }
end
```

#### Notes

1. generate elastichsearch trong job `Cache::GenerateElasticsearchDataJob`
2. xem thông tin hàm `to_hash` trong model field và block để biết được sẽ đẩy thông tin gì vào elastichsearch
3. xem `AssetRepository` để biết cách mapping elastichsearch
4. xem `Assets::ElasticsearchService` để biết câu query elastichsearch ntn

### Map Page

kiến thức để làm phần này:

- ruby
- javascript es6
- filter with ES
- 1 số kiến thức về ReactJS

page này gồm các phần chính:

- backend `Maps::BuildJsonService`
- frontend filter, vẽ map và các action với map

#### Backend

mục tiêu là query thông tin database ra thành 1 file json trả về cho FE để show, nhưng trong quá trình làm việc thì có 1 số issue ( thời gian query và generate ra file json chậm, issue cache, code smell) nên solution là thay vì mỗi lần query và trả về data cho FE xử lí thì hầu hết việc xử lí đều ở backend, sau đó sẽ ghi vào 1 file json và up lên s3, mỗi khi FE request thì sẽ lấy url của file json cache map này và cho FE tải về

chú ý:

- file json này sẽ đã được add vào cronjob mỗi ngày generate 1 lần
- mỗi lần đưa trả đường link đều có token riêng nên sẽ không bị vấn đề browser cache
- nếu có vấn đề về tốc độ sau này thì nên dùng cdn hoặc biện pháp khác để giúp khách hàng tải file này về nhanh hơn ( hiện bucket đang ở west-2 oregon, nên suy nghĩ sau này sẽ scale ra các region khác)

luồng chạy:

Customer::MapsController#search -> Maps::BuildJsonService -> handlers -> facades

Maps::BuildJsonService -> service sẽ giúp gọi những handler tương ứng cho từng layer, nếu có nhu cầu thêm/bớt layer thì nên xử lí trong này
handlers -> mỗi layer đều có handler riêng ( FieldHandler, BlockHandler, WellHandler ), tuỳ vào mỗi nước mà sẽ có điều kiện để query ra các resource này khác nhau

vd: với nước bahamas thì có điều kiện query khác và includes khác nhau

```ruby
def find_well
  if country.id == BAHAMAS_ID
    Well.where(country: country)
        .with_coordinate
        .select(ATTRIBUTES_FOR_THE_MAP)
  else
    Well.where(country: country)
        .where.not(name: DISABLED_WELL)
        .with_coordinate
        .includes(field: :basin, well_block: :basin, block: :basin)
        .select(ATTRIBUTES_FOR_THE_MAP)
  end
end
```

facades -> sau khi đã query được resource với những điều kiện tương ứng thì sẽ đẩy record đó vào trong facade layer để render json cho phù hợp, tầng này sẽ hỗ trợ xử lí thông tin để FE có thể render dễ dàng nhất

```ruby
def to_json(*_args)
  {
    name: name_anp,
    field_name: field_name,
    well_type: well_type,
  ...
  }
end

def well_type
  well.cached_map_data['map_data'] && well.cached_map_data['map_data']['well_type']
end
```

sau khi ruby đã generate ra được 1 object data chứa các layer ( trong class `Maps::BuildJsonService` ) thì sẽ ghi ra file json và up lên s3 bucket ( đọc `Maps::CachingService` ), khi FE request thì sẽ lấy link của file json này đưa cho FE tải về

#### Frontend

FE trên map gồm các phần chính:

- request BE để lấy thông tin filter cho đúng
- request BE để lấy link file json ở trên
- sau khi đã lấy file json ở trên thì vẽ map dựa vào data trong file json
- sau khi đã vẽ map và các layer, user có thể filter và select 1 số field/block/facility

##### Request BE để lấy thông tin filter cho đúng

xem file `app/assets/javascripts/index.js`

```
GoogleMap.loadCountryRelativeData(value, { on_map: true });
```

##### request BE để lấy link file json ở trên

```js
function reRenderMaps(options)
```

##### Vẽ map với thông tin vừa nhận

```js
GoogleMap.initCircleMap(data, clearMarkers)
```

file này viết bằng Reactjs

##### user có thể filter và select

user filter data trong FE, vd chọn company nào đó => chỉ hiện những data liên quan đến company đó

```
app/javascript/components/Maps/MapScripts/filterMaps.js
```

user khi click vào 1 asset ( block/field ) sẽ zoom vào asset đó

```
app/javascript/components/Maps/MapScripts/mapHelpers/focusMap.js
```

đồng thời sẽ hiện popup thông tin của object đó

```
app/javascript/components/Maps/MapScripts/infoWindows/index.js
```

ngoài ra còn 1 số tính năng khác như vẽ chart cho facility ( `app/javascript/components/Maps/MapScripts/mapHelpers/drawChart.js` ), chọn field nhưng zoom vào block ...

### Run R

đầu tiên cần clone code của các repo ML về các folder nước tương thich

vd:

- https://github.com/welligence/BrazilML -> brazil
- https://github.com/welligence/ArgML -> argentina

Install R với version `3.4.3` cho giống với môi trường staging
tạo file dump country level cho nước cần chạy

```ruby
DumpDataJob.perform_now('Brazil') # generate cho nước Brazil
```

sau khi dump xong, kiểm tra trong folder `/dump_data/brazil` xem có file dump chưa

```s
➜ ls -l dump_data/brazil
total 731416
-rw-rw-r-- 1 ubuntu ubuntu 114023183 Jul 15 00:19 Brazil_data.rds
-rw-rw-r-- 1 ubuntu ubuntu    145776 Jul 15 00:17 gas_prices.csv
-rw-rw-r-- 1 ubuntu ubuntu   2539984 Jul 15 00:17 gas_statistics.csv
-rw-rw-r-- 1 ubuntu ubuntu   5071437 Jul 15 00:16 hist_field_prod.csv
-rw-rw-r-- 1 ubuntu ubuntu    841216 Jul 15 00:17 inputs.csv
-rw-rw-r-- 1 ubuntu ubuntu    163902 Jul 15 00:17 oil_prices.csv
-rw-rw-r-- 1 ubuntu ubuntu 408757342 Jul 15 00:15 production_data.csv
-rw-rw-r-- 1 ubuntu ubuntu     84410 Jul 15 00:17 schedule_inputs.csv
-rw-rw-r-- 1 ubuntu ubuntu     30598 Jul 15 00:16 sim_assets.csv
-rw-rw-r-- 1 ubuntu ubuntu   3961054 Jul 15 00:15 well_data.csv
-rw-rw-r-- 1 ubuntu ubuntu 212168973 Jul 15 00:20 well_injection_data.csv
-rw-rw-r-- 1 ubuntu ubuntu   1136331 Jul 15 00:15 well_type.csv
```

tiến hành chạy R1, R2

```ruby
GeneratePredictionsFieldJob.perform_now('Brazil', 14, false) #run R1
GenerateR2OutputsFieldJob.perform_now('Brazil', 14, false) #run R2
```

_flow run R gồm các step_:

- symlink dump data file vào folder data process của asset đó
- generate dump file asset level
- chạy R script tương ứng ( R1, R2, Rfinal, ... trong bước này gồm các bước nhỏ như: copy r code về data processor, chạy rscript, nếu có error gì thì ghi ra log file )
- kiểm tra xem có error không bằng cách xem log ( xem trong `RCommon`, `R1ScriptService`, ... )
- nếu có error thì update r_status là failed
- nếu không có error thì bắt đầu bước parse data output của R1/R2/R final
- trong bước parse data sẽ kiểm tra có đủ file output không, nếu không có đủ sẽ báo missing và error
- import file output vào trong db để xử lí sau này

```ruby
  def parse_r_output
    OutputCsv::PredictionService.new(data_processor).perform
  end
```

note:

- có thể output của R1 là input của R2
- có thể output của R2 chính là input của R1 lần tiếp theo chạy

### Generate Excel Report

#### Flow generate excel report

sau khi đã chạy đc R1, R2, Rfinal... đủ điều kiện và data thì ta có thể dùng data đó để generate ra file excel asset model

```ruby
GenerateReportFieldJob
GenerateReportBlockJob
```

original asset là asset được tạo thông qua quá trình import shapefile, excel, những asset này đã đủ điều kiện để có production.
new asset là những asset mới, chưa bắt đầu sản xuất được
mỗi loại lại có các kiểu `fiscal_regime` khác nhau, phổ biến là `Concession` và `PSC`

vd với nước Brazil thì có 4 loại tương ứng với 4 loại template

- original asset Concession
- original asset Psc
- new asset Concession
- new asset Psc

sau khi đã biết được asset loại nào thì ta cần tìm generator tương ứng với asset đó

```ruby
def generator
  country_name = field.country.name.split(' ').first.titleize
  begin
    "Report::#{country_name}::Excel::#{field.field_template_type}::#{field.fiscal_regime_type}::Generator".constantize
  rescue StandardError => _e
    # TODO: Implement other country to match above pattern
    case country_name
    when 'Suriname'
      "Report::#{country_name}::Excel::#{field.fiscal_regime_type}::Generator".constantize
    when 'Argentina'
      Report::Generator
    else
      "Report::#{country_name}::ManageGenerator".constantize
    end
  end
end


=> generator.new(field, self).perform
```

mỗi file report excel sẽ có nhiều sheet ( Cover, Asset Overview, Asset Detail,...) nên ta cũng chia ra thành nhiều class component, mỗi class đảm nhận 1 sheet gọi là `sheet_modifiers`

cấu trúc folder của 1 loại report

```s
├── generator.rb
├── sheet_modifiers
│   ├── asset_detail.rb
│   ├── asset_overview.rb
│   ├── base_modifier.rb
│   ├── cash_flows.rb
│   ├── charts.rb
│   ├── comps.rb
│   ├── cover.rb
│   ├── depreciation.rb
│   ├── hist_drilling.rb
│   ├── inputs.rb
│   ├── reserves.rb
│   ├── revenue.rb
│   ├── total_prod.rb
│   ├── type_curve.rb
│   └── well_by_well.rb
└── sheet_modifiers.rb
```

vd muốn update cho sheet `Input` thì ta vào trong file `sheet_modifiers/inputs.rb` để update code

việc sẽ xử lí sheet nào dựa vào hàm perform của generator đó

```ruby
MODIFIERS = %w[
  Cover
  AssetOverview
  AssetDetail
  Charts
  Inputs
  WellByWell
  TypeCurve
  Revenue
  Reserves
  Depreciation
  TotalProd
  CashFlows
  HistDrilling
  Comps
].freeze

def perform
  prefix = Report::Brazil::Excel::OriginalAsset::Concession::SheetModifiers
  MODIFIERS.each do |sheet_name|
    sheet = workbook[SHEETS[sheet_name.underscore.to_sym]]

    "#{prefix}::#{sheet_name}".constantize.new(field, sheet).perform
  end
end
```

vd sau này cần phải thêm sheet Abc thi ta sẽ:

- tạo file `abc.rb` trong folder `sheet_modifiers` và implement phần insert dữ liệu
- thêm `Abc` vào trong `MODIFIERS`
- kiểm tra trong `SHEETS` đã có sheet tên `Abc` chưa...

Notes: cần kiểm tra đã có sẵn file template tương ứng với loại asset đang generate chưa, thường sẽ đặt trong folder `asset-model-templates`

```ruby
def excel_template
  file_path = 'asset-model-templates/brazil/'
  file_path += asset_type.titleize.parameterize
  file_path +=
    if field.name == 'SUL DE SAPINHOA'
      '-concession'
    else
      case field_type
      when 'Concession' then '-concession'
      when 'Transfer of Rights' then '-tor'
      when 'Production Sharing Contract' then '-psc'
      else '-concession'
      end
    end

  template = Rails.root.join("#{file_path}.xlsx")
  raise Exceptions::AssetModelTemplateNotExists unless template.exist?

  template
end
```

#### Implement cho từng sheet_modifiers

dùng thư viện `rubyxl` để đọc và tương tác với các file excel

Notes:

- hầu hết các hàm đã được implement trong các module Reportable, BlockReportable, Report::Utilable, ...
- sau này khi có 1 hàm nào cần implement thì nên move vào các module, vd cho `AssetOverview` => `Brazil::AssetOverviewable`, nếu có nhiều nước xài thì move hàm đó vào cái chung vd `AssetOverviewable`
- tham khảo code từ những sheet cũ đã implement và follow theo style code

### Generate Enhanced Report PDF

đầu tiên, gọi là enhanced report bởi vì trước đó đã có 1 bản pdf report, nhưng bản đó chỉ là version đơn giản, ít thông tin. Sau này khách hàng có nhu cầu nâng cấp report lên 1 phiên bản nhiều thông tin hơn nên sinh ra enhanced report

phần này gồm 2 step:

- generate ra enhanced excel report ( 1 file excel )
- dựa vào file excel vừa được generate để generate ra bản report pdf

check file `GenerateEnhancedReportFieldJob`

```ruby
def generator
  country_name = field.country.name.split(' ').first.titleize
  begin
    "Report::#{country_name}::EnhancedExcel::#{field.field_template_type}::#{field.fiscal_regime_type}::Generator".constantize
  rescue StandardError => _e
    case country_name
    when 'Peru'
      'Report::Peru::EnhancedExcel::Generator'.constantize
    when 'Suriname'
      "Report::#{country_name}::EnhancedExcel::#{field.fiscal_regime_type}::Generator".constantize
    else
      "Report::#{country_name}::EnhancedExcel::Generator".constantize
    end
  end
end
```

#### Generate enhanced excel report

flow: upload excel report to `M$ Sharepoint` -> pull data -> generate enhanced excel report -> upload to `S3`

dựa vào những thông tin từ R/Scrape data/Manual input hoặc từ Excel Report mà ta sẽ import thông tin vào file template excel , xem trong folder `enhanced_template`, hầu hết các nước đều dùng default `template.xlsx`, với ARG, GOM thì có custom lại nên dùng template khác ( client up lên trello )

```s
enhanced_template
├── argentina_template.xlsx
├── bolivia_template.xlsx
├── gom_template.xlsx
└── template.xlsx
```

tập trung vào file `Generator` của các nước trong module `EnhancedReport`

```ruby
module Report
  module Brazil
    module EnhancedExcel
      module OriginalAsset
        class Concession::Generator < Generator
```

cấu trúc sourcecode cũng tương tự bên module `Report`

```s
├── generator.rb
└── sheet_modifiers
    ├── base_modifier.rb
    ├── costs.rb
    ├── development.rb
    ├── facilities.rb
    ├── financials.rb
    ├── production.rb
    ├── project_high.rb
    └── reserves.rb
```

module `Report::Template` là phần dùng chung của tất cả các `EnhancedReport::Generator`, module này rất quan trọng trong step này, nên xem kĩ các bước của nó

Lưu ý: module này có sử dụng multi thread để tăng tốc độ upload/pull data từ `M$ Sharepoint`, cần xem máy mình có chịu nổi ko =))), nên set từ 4-8 thread là hợp lý, ở trên staging thì cứ set 1-2 thread là ok rồi

#### Generate enhanced pdf report

flow: upload enhanced excel report -> tổng hợp data theo từng sheet -> generate ra file html -> convert html to pdf bằng gem `WickedPDF`

dựa vào thông tin của enhanced excel, chart trong đó mà ta sẽ generate ra file html đầy đủ thông tin và biểu đồ cho khách hàng hơn

chú ý module `Report::EnhancedPdfable`

file enhanced report template `layouts/enhanced_pdf.haml`

```s
├── generator.rb
└── sheet_pullers
    ├── analytics.rb
    ├── asset_overview.rb
    ├── base_puller.rb
    ├── costs.rb
    ├── development.rb
    ├── facilities.rb
    ├── facility_production.rb
    ├── financials.rb
    ├── map.rb
    ├── ownership.rb
    ├── production.rb
    ├── project_high.rb
    ├── reserves.rb
    ├── risk_opp.rb
    └── start.rb
```

cấu trúc thư mục và class cũng tương tự như module `report` và `enhanced report`, cũng chia ra thành các tab

```ruby
module Report
  module Brazil
    module EnhancedPdf
      module OriginalAsset
        module Concession
          class Generator
            include Report::EnhancedPdfable
```

Lưu ý: module này có sử dụng multi thread để tăng tốc độ upload/pull data từ `M$ Sharepoint`, cần xem máy mình có chịu nổi ko =))), nên set từ 4-8 thread là hợp lý, ở trên staging thì cứ set 1-2 thread là ok rồi

### Cron Job

Hệ thống sẽ có những job cần được thiết lập để chạy vào 1 số giờ nhất định, vd: Gửi report vào cuối tuần, clear database vào cuối tháng, ... => Cron job

trong dự án welligence thì sử dụng gem `whenever`

Lưu ý:

- Cái nào chạy lâu thì bỏ vào job
- Cái nào job nên chạy cố định vào một thời điểm thì xài cronjob => whenever

## Server Config

### ML code

- setup trên server staging, production tương tự như ở dưới local, sẽ đặt code R trong thư mục `machine_learning`
- khi clone code về thì đặt tên lại folder thành tên nước ( BrazilMl -> brazil, MexML -> mexico)
- folder `WellML` là folder chứa code package của dự án do chính team DS của welligence xây dựng lên, có thể cài đặt bằng 2 cách là :

* Run file `runme_server.R`
* Rscript -e "devtools::install_github('sethneel/WellML', auth_token = '$AUTH_TOKEN', upgrade = 'never', force = TRUE, ref = '$WELLML_BRANCH')"

### Backup Database

Welligence sử dụng dịch vụ RDS của AWS, mỗi ngày sẽ tự snapshot lại, limit là 7 ngày

Ngoài ra để giúp dev có thể có bản backup thì còn có sử dụng gem `backup` để dev có thể chọn database nước nào cần để backup ( xem `BackupWorker` )

Gem `backup` được config trong folder `/home/ubuntu/Backup`

```s
├── angola_backup.rb
├── argentina_backup.rb
├── bahamas_backup.rb
├── barbados_backup.rb
├── bolivia_backup.rb
├── brazil_backup.rb
├── chile_backup.rb
├── colombia_backup.rb
├── cuba_backup.rb
├── dominican_republic_backup.rb
├── ecuador_backup.rb
├── falkland_islands_islas_malvinas_backup.rb
├── french_guiana_backup.rb
├── ghana_backup.rb
├── gom_backup.rb
├── guyana_backup.rb
├── honduras_backup.rb
├── jamaica_backup.rb
├── master_backup.rb
├── mexico_backup.rb
├── nicaragua_backup.rb
├── nigeria_backup.rb
├── peru_backup.rb
├── production_backup.rb
├── suriname_backup.rb
├── trinidad_and_tobago_backup.rb
├── uruguay_backup.rb
└── venezuela_backup.rb
```

khi dev vào trang `https://staging.welligence.com/admin/countries`, chọn nước và bấm backup => tạo job `BackupWorker` cho nước đó và run script backup, sau khi backup thành công sẽ up lên s3 và gửi mail backup success

Note:
các job backup sau này cũng hoạt động tương tự, chỉ thay `bucket name` và `limit`

## Jenkins

Welligence có config jenkins dùng để cho việc CI/CD

### Run Job

có 1 số job thường xuyên chạy như:

- http://35.155.0.83:8080/job/deploy-branch-to-env -> deploy branch
- http://35.155.0.83:8080/job/run-r -> run r trên ml

sau khi chọn job cần chạy thì bấm vào `Build with Parameters` và chọn input phù hợp ( vd chọn nước, chọn branch, ...)

### Config Job

script của jenkins nên được backup vào repo này `https://github.com/welligence/WellJenkins`, sau này khi có nhu cầu update gì thì sẽ push code vào repo này, sau đó ssh lên con jenkins và pull code về, giúp cho code CI có thể revertable

trong `README.md` đã bao gồm hướng dẫn để start jenkins cổng 8080

#### Khi có nhu cầu update config job

1. chọn job cần update
2. click Configure
3. thêm input hoặc xoá input cũ => Add Parameter
4. Update excute code nếu cần
5. Bấm Save
6. Double check

## WorkerConfig

We have about 30 EC2 instances to serve R code/generate asset model with prefix [brazil_welligence_ml]. Each instance is an app that have a sidekiq worker listen to a redis host which locate in [prefix]\_1 instance (Eg: brazil_welligence_ml_1).
What we need to do here are: - Update database connection in database.yml - Update bucket name in application.yml - Update Redis host in application.yml - Check out to correct branch of Brazil/Mexico/Peru machine learning repository - Check out to correct branch for main repository

```
1. **run_r.sh**:
   - Update main repository branch to `master`, `staging`, or whatever.
   - Update machine_learning repositories branch (Brazil, Mexico).
   - Install dependencies for Rails application.
   - Restart sidekiq for background jobs.
   - Sync dump data folder for R code from correct server (staging/prelive/live)
2. **welligence_shutdown.sh**:
   - Sync data from instances to correct server after R code finish (actually after the instance stop).
3. **application.yml**:
   - Store environment variables for Rails application.
   - We need to change somethings here (Eg: **redis_host**, **s3_bucket_name**)
4. **database.yml**:
   - Store database connection information.
   - We need to change the **host_name** here to correct rds **host_name**.
```

có 2 cách để update config trên Mls

### Local Config

#### Get dynamic IPs for ML instances the save to ~/.ssh/config

1. Create `~/.ssh/welligence/ec2.d`
2. Add `Include ~/.ssh/welligence/ec2.d` to the beginning of `~/.ssh/config` file
3. Run ruby code to get dynamic IPs for ML instances

- install gem `ec2-writer` # if needed
- Go to rails console and run

```
require 'ec2/writer'
Ec2::Writer::Runner.new.write_config(file_name = 'ec2.d', host_prefix = 'brazil_welligence_ml') # You can change to mexico if needed
```

4. Check file `~/.ssh/welligence/ec2.d` if it has IP config. If not please check config

#### Config to R for a country

1. Start EC2 ML instances with correspond country (Eg: `brazil_welligence_ml_*` for Brazil)
2. Update all information with `TODO: tag` in:

- config/application.yml: s3_bucket_name, redis_host
- config/database.yml: host
- welligence_shutdown.sh: destination_server_ip is `staging_ip` or `live_ip` or `prelive_ip`
- run_r.sh: repo_branch, brazil_branch, mexico_branch, peru_branch, colombia_branch, argentina_branch and destination_server_ip
- prepare_server.sh: server_prefix, root_dir (You have to config ssh host for ML instances first - which was defined in above step) ( my is `root_dir="/Users/Ken/Sourcecode/WellStack"`)

3. Stop and Start ML instances ( dont Reboot )
4. SSH to brazil_welligence_ml_1 and run R for all active fields of country. (how to run is defined before in README).

### Jenkin Config

mặc định là khi chạy job run r trên jenkin sẽ làm các bước sau:

- update file application.yml, database.yml dựa vào input
- mở Mls, scp những file config này lên
- tắt và mở lại Mls để trigger chạy những file vừa up lên
- Run R, Generate Report

đến bước thứ 3 xong là ta đã update được config của Mls

## Release Step

sẽ có 2 loại release: major và minor. major khi có các update lớn, hoặc qua 1-2 tháng thì nên major release 1 lần. minor release là khi Bussiness team request release/fix bug 1 feature nào đó lên live

### Major version

client yêu cầu release 1 nước nào đó, hoặc có update lớn ảnh hưởng source code hoặc đã quá nhiều minor version thì nên release major

các step release

vd hiện tại đang có nhánh 2.45.15 trên live

```
- checkout qua nhánh 2.46.0
- merge các release branch của các nước cần release vào
- update file `version.html`
- backup và restore những nước cần release từ staging lên live
- copy S3 folder của các nước đó
- scp folder dump và folder data_proccess của các nước đó lên live, lưu ý nên có thể revertable
- deploy bản 2.46.0 lên live
- clear cache, generate new data
- request client check
- sau khi client đã check và confirm thì từ các nhánh release country ( vd 2.46_bra ), checkout ra nhánh release country mới ( 2.47_bra ), sau đó merge với release master ( 2.46.0 )
```

### Minor version

release minor cũng tương đối giống release major

vd hiện tại đang có nhánh 2.45.15 trên live

```
- checkout ra nhánh 2.45.16
- merge release branch của nước cần release/hot fix
- update file `version.html`
- backup và restore những nước cần release từ staging lên live
- copy S3 folder của các nước đó
- scp folder dump và folder data_proccess của các nước đó lên live, lưu ý nên có thể revertable
- deploy bản 2.45.16 lên live
- clear cache, generate new data
- request client check
- sau khi client đã check và confỉm thì merge nhánh 2.45.16 vào nhánh 2.46.0, sau đó merge nhánh 2.46.0 vào tất cả các nhánh release của các nước
```

nếu release chỉ bao gồm code, không có data thì có thể skip bước này

```
- backup và restore những nước cần release từ staging lên live
- copy S3 folder của các nước đó
- scp folder dump và folder data_proccess của các nước đó lên live, lưu ý nên có thể revertable
```

## Fix Template

Các step để fix template có externalLink:

- unzip Ghana\_-_V2.xlsx -d data -> unzip file template ra folder data
- vào folder data, tìm và xóa externalLink trong mấy file html
- xóa trong workbook.xml.res va workbook.xml
- tìm những chỗ liên quan đến cái externalLink mà xóa đi ( ghana bị lỗi do có liên quan đến externallink )
- zip -r v1.xlsx \* ( đứng trong folder data ) -> zip lại thành file template v1.xlsx
- move qua máy windows, mở thì thấy có lỗi -> cho windows sửa lỗi và save lại thành v2.xlsx
- dùng v2 làm template để generate + có bật excelize golang -> fix đc lỗi khi vừa open, nhưng vẫn còn lỗi khi mở tab revenue
- mở v2 trên mac, save lại thành v3.xlsx
- dùng v3 để generate ra -> hiện tại ko còn lỗi hiện popup warning nữa, chắc đã fix đc rồi, đã có thể mapping tiếp

## Create New Country

Khi estimate 1 card cho new country cần hỏi cus về structure của country đó sẽ giống với country nào trong database mình.

Các step để làm card này:

1. ssh lên `Staging` và `Live`
  - connect vào master database -> create 1 record cho country này (Staging và Live phải cùng id), country model đã add `SyncDatabaseJob` vào nên nó sẽ tự sync data sang tất cả các database có trong shard config
  - create 1 database từ `welligence_sample` template với name là `welligence_[country_name]`
2. checkout từ master và làm bình thường, trong step này sẽ bao gồm việc config country level (static_country_properties.yml, static_country_dropdown_values.yml, ...)
Note: khi deploy lên `Staging` nhớ connect vào database của country này để add static_country_properties
`RakeTask::StaticCountryProperties::AddMissingPropertiesJob.perform_now(country_name)`
