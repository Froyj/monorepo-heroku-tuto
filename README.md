# Deployment procedure with monorepo

Repo structure 
```
.
├── back
│   └── package.json
├── front
│   └── package.json
├── package.json
└── postbuild.sh
```

## Create the app

``` bash
heroku apps:create --app <appname> --region eu
```

## Attach ClearDB database 
```
heroku addons:create cleardb:ignite
```

## Get ClearDB database URL
```
heroku config:get CLEARDB_DATABASE_URL
```

you'll get something like this :
```
mysql://b4ad2124eabed2:f3zace00@eu-cdbr-west-02.cleardb.net/heroku_4dbbbcd1b203325?reconnect=true
```

where 

DATABASE_USER=b4ad2124eabed2

DATABASE_PASSWORD=f3zace00

DATABASE_HOST=eu-cdbr-west-02.cleardb.net

DATABASE_NAME=heroku_4dbbbcd1b203325


## Setup environment variables for our app
```
heroku config:set DATABASE_USER=b4ad1146eabed2 DATABASE_PASSWORD=f3efce00 DATABASE_HOST=eu-cdbr-west-02.cleardb.net DATABASE_NAME=heroku_4dbbaad1b203325
```

## Feed the database with your schemas and data

To connect your database you can use Workbench or your MySQL CLI tool and configure connection with the informations above.

Connection with CLI :

```
mysql -h eu-cdbr-west-02.cleardb.net -u b4ad2124eabed2 -p -D heroku_4dbbbcd1b203325
```
and enter your password :

```
f3efce00
```

To import your .sql files :

```
mysql -h eu-cdbr-west-02.cleardb.net -u b4ad2124eabed2 -p -D heroku_4dbbbcd1b203325 < mySqlSchemas.sql
```

## Replace values in our conf.js with env variables 

```
const mysql = require('mysql');

const { DATABASE_HOST, DATABASE_USER, DATABASE_PASSWORD, DATABASE_NAME } = process.env;

const connection = mysql.createConnection({
  host: DATABASE_HOST,
  user: DATABASE_USER,
  password: DATABASE_PASSWORD,
  database: DATABASE_NAME
});

module.exports = connection;
```




**Don't forget to remove conf.js from .gitignore once this is done.**
## Add scripts for deployment

In our package.json, we have to add those scripts (heroku-build and start)

```
"scripts": {
    "heroku-postbuild": "bash ./postbuild.sh",
    "start": "cd back && node index",
    "start:dev": "npx lerna run start --stream",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
```
We create the postbuild.sh script called via heroku-postbuild script :

```
# postbuild.sh

exit_on_error() {
  exit_code=$1
  if [ $exit_code -ne 0 ]; then
    >&2 echo "build command failed with exit code ${exit_code}."
    exit $exit_code
  fi
}

cd back
npm install
echo "installing back dependencies"
cd ../front
npm install
echo "installing front dependencies"
npm run build
echo "building for production"

```

## Serve our app from backend

We our app is started only backend is running. Then we have to make it serve our front-end app.

we will need those 2 modules in ou index.js 

```
const connection = require('./conf');
const path = require('path');
``` 


In our index.js at the beginning we want to add these lines in order to serve our front-end index.html file. 

```
app.use(express.static(path.resolve(__dirname, '../front/build')));

app.get('/', (req, res) => {
  res.sendFile(path.resolve(__dirname, '../front/build', 'index.html'));
});
```

## Config port for backend

index.js
```
const port = process.env.PORT || 5000;
```

## Prevent lost of database connection with our app

### The dirty way

Just above this
```
app.use(express.static(path.resolve(__dirname, '../front/build')));
```
Add these lines in order to make a recurrent call to the database and keep connection alive
```
setInterval(() => {
  connection.query('SELECT 1;');
}, 6000);
```

### The clean way

Doing this in a clean fashion require to make use of a [pool config](https://github.com/mysqljs/mysql#pooling-connections)



## Deploy our app on heroku 

Finally we want to push our app for deployment

```
git push heroku master
```