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
    - [Generate Enhanced Report](#generate-enhanced-report)
    - [Cron Job](#cron-job)
  - [Server Config](#server-config)
    - [ML code](#ml-code)
    - [Backup Database](#backup-database)
  - [Jenkins](#jenkins)
    - [Run Job](#run-job)
    - [Config Job](#config-job)
  - [Monitor](#monitor)
    - [Sidekiq](#sidekiq)
  - [Worker Config](#server-config)
    - [Local config](#local-config)
    - [Jenkin config](#jenkin-config)
  - [Release Step](#release-step)
    - [Major version](#major-version)
    - [Minor version](#minor-version)
  - [Add New Country](#add-new-country)

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
- mỗi nước sẽ có 1 db shard chứa data của riêng nước đó mà thông, vd db `welligence_brazil` chỉ chứa thông tin của riêng Brazil mà thôi, tương tự với `welligence_mexico`, `welligence_colombia`

việc config sharding này dựa vào column `db_name` trên table `countries` trên db master

```
#<Country:0x00007f9c35f6d9d8 id: 1, name: "Brazil", db_name: "welligence_brazil">
#<Country:0x00007f9c35f6d7a8 id: 3, name: "Mexico", db_name: "welligence_mexico">
```

NOTE: Brenton đang có dự tính chuyển từ dynamic config dựa vào table `countries` và thay bằng hard config trong file `shard_configuration.rb`. Update lại khi đã đc merge

## Workflow

do welligence là 1 app với nhiều country, data đã chia ra thành các database riêng nên gitflow cũng sẽ được chia thành các nhảnh nhỏ
sẽ gồm các nhánh: master, staging, release, .... trong đó:

- master: là nhánh chứa code đã được test trên staging, live
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
