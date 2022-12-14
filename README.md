# Tool for Integration Testing NestJS using Mongodb Replica Set 

[![Mongoose](https://img.shields.io/badge/MongoDB-4EA94B?style=for-the-badge&logo=mongodb&logoColor=white)](https://github.com/Automattic/mongoose)
![stars](https://img.shields.io/github/stars/Automattic/mongoose?style=social)

[![NestJs](https://img.shields.io/badge/nestjs-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)](https://github.com/nestjs/nest)
![stars](https://img.shields.io/github/stars/nestjs/nest?style=social)

## Background
Backend services built on top of [NestJS framework](https://nestjs.com/) is able to utilize the test framework [Jest](https://docs.nestjs.com/fundamentals/testing). NestJS and Jest come up with solutions to develop unit tests and end-to-end tests. While both are necessary, they are out of scope of this document. Here we present a documentation for integration tests, which is where the tests are run against a service integrated with its dependencies, e.g. database, cloud features, etc.

### Situation
Previously tests in my project were done manually by QA or Product Ops. For simple cases, this practice is enough. However, the project is growing and now has advance features for business process. Advance features are hard to be tested by detail in manual test, because it is exhausting and prone to human error. Furthermore, QA or Product Ops are handed with other daily tasks that takes time.

Another problem is coming from service dependencies which have advance features. For example, Mongo DB with its Node.js library called mongoose has a “Transaction” feature that is really important for data consistency. Me and my team have experienced data inconsistency in the mid of 2022. QA and Product Ops found the tests are passed in manual test. However, in production inconsistency is still exist. 

### Task

Manual test can’t be used in advance engineering problem. So, test automation is a way to brake this hard issue. Problems coming from dependencies can’t be tested in unit test, yet it is overkill to be tested in end-to-end test. That’s why engineer offer a solution to create integration test.

### Action
Engineer prepare requirements and steps to do integration test in backend services.

## Result

### Requirements
1. Shell scripts containing mongodb replica set initiation placed in root with directory `scripts/setup.sh`. Here pre-defined script for up to 3 replica set with 1 primary.
```sh
#!/bin/bash

#MONGODB1=`ping -c 1 mongo1 | head -1  | cut -d "(" -f 2 | cut -d ")" -f 1`
#MONGODB2=`ping -c 1 mongo2 | head -1  | cut -d "(" -f 2 | cut -d ")" -f 1`
#MONGODB3=`ping -c 1 mongo3 | head -1  | cut -d "(" -f 2 | cut -d ")" -f 1`

MONGODB1=mongo_rs1
# MONGODB2=mongo_rs2
# MONGODB3=mongo_rs3

echo "**********************************************" ${MONGODB1}
echo "Waiting for startup.."
until curl http://${MONGODB1}:27017/serverStatus\?text\=1 2>&1 | grep uptime | head -1; do
  printf '.'
  sleep 1
done

# echo curl http://${MONGODB1}:28017/serverStatus\?text\=1 2>&1 | grep uptime | head -1
# echo "Started.."

for n in $(seq 1); do
  until mongo --host "mongo_rs${n}" --eval "print(\"waited for connection\")"; do
      echo -n .; sleep 2
  done
done


echo SETUP.sh time now: `date +"%T" `
mongo --host ${MONGODB1}:27017 <<EOF
var cfg = {
    "_id": "rs0",
    "protocolVersion": 1,
    "version": 1,
    "members": [
        {
            "_id": 0,
            "host": "${MONGODB1}:27017",
            "priority": 2
        }
    ],settings: {chainingAllowed: true}
};
rs.initiate(cfg, { force: true });
rs.reconfig(cfg, { force: true });
rs.slaveOk();
db.getMongo().setReadPref('nearest');
db.getMongo().setSlaveOk(); 
EOF
```

2. Docker-compose file placed in root, `docker-compose.yml`. This file will create mongodb replica set containers and one db setup container. 
```yaml
version: "3.4"
services:
  test-db-setup:
    container_name: test-db-setup
    image: mongo
    restart: on-failure
    networks:
      default:
    volumes:
      - ./scripts:/scripts
    entrypoint: [ "sh", "/scripts/setup.sh" ]
    depends_on:
      - mongo_rs1
      # - mongo_rs2
      # - mongo_rs3
  
  mongo_rs1:
    hostname: mongo_rs1
    container_name: localmongo_rs1
    image: mongo
    expose:
      - 27017
    ports:
      - 27017:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0", "--journal", "--dbpath", "/data/db" ]
    volumes:
      - ./mongo/data_rs1/db:/data/db # This is where your volume will persist. e.g. VOLUME-DIR = ./volumes/mongodb
      - ./mongo/data_rs1/configdb:/data/configdb
  # mongo_rs2:
  #   hostname: mongo_rs2
  #   container_name: localmongo_rs2
  #   image: mongo
  #   expose:
  #     - 27017
  #   ports:
  #     - 27018:27017
  #   restart: always
  #   entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0", "--journal", "--dbpath", "/data/db" ]
  #   volumes:
  #     - ./mongo/data_rs2/db:/data/db # Note the data_rs2, it must be different to the original set.
  #     - ./mongo/data_rs2/configdb:/data/configdb
  # mongo_rs3:
  #   hostname: mongo_rs3
  #   container_name: localmongo_rs3
  #   image: mongo
  #   expose:
  #     - 27017
  #   ports:
  #     - 27019:27017
  #   restart: always
  #   entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "rs0", "--journal", "--dbpath", "/data/db" ]
  #   volumes:
  #     - ./mongo/data_rs3/db:/data/db
  #     - ./mongo/data_rs3/configdb:/data/configdb
```

3. Npm scripts for pretest, test, and posttest in `package.json`.
```json
{
  "scripts": {
    ...,
    "test:integration": "sleep 5 && jest --detectOpenHandles --config ./test/jest-integration.json",    
    "pretest:integration": "docker-compose up -d test-db-setup",
    "posttest:integration": "docker-compose stop test-db-setup && docker-compose rm -f test-db-setup && docker-compose down",
  }
} 
```

4. Create a file inside test directory, `test/jest-integration.json`, to enable jest identify test cases for integration test.
```json
{
  "moduleDirectories": [
    "<rootDir>/../", 
    "node_modules"
  ],
  "moduleFileExtensions": ["js", "json", "ts"],
  "testEnvironment": "node",
  "testRegex": ".integration.spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  }
}
```

5. `.env` file placed at the root of the project containing the `DB_URI_TEST`
```.env
DB_URI_TEST=mongodb://localhost:27017/db-test?readPreference=primary&directConnection=true
```

6. Create test cases inside file name with pattern `<any case>.integration.spec.ts` in test directory. For example:
```
test/order.integration.spec.ts
test/product.integration.spec.ts
```
Make sure to connect to the primary host before all test cases.
```typescript
import { INestApplication } from '@nestjs/common';
import { Test, TestingModule } from '@nestjs/testing';
import { connection, Connection, Types } from 'mongoose';
import { SomethingController } from 'src/something/something.controller';
import { AppModule } from '../src/app.module';

describe('SomethingController', () => {
  let app: INestApplication;
  let dbConnection: Connection;
  let contoller: SomethingController;
  
  beforeAll(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      imports: [
        AppModule,
      ],
    }).compile();

    app = await moduleRef.createNestApplication();
    await app.init();

    console.log('app initated!');

    controller = app.get<SomethingController>(SomethingController);

    // connect to mongodb database primary host of the replica set
    const configService: ConfigService = app.get(ConfigService);
    if (!configService.get('DB_URI_TEST')) {
      throw new Error(`Must define env var "DB_URI_TEST"`);
    }
    await connect(configService.get('DB_URI_TEST'));
  }, 50000);
  
  afterAll(async () => {
    await dbConnection.close();
    await app.close();
  }, 20000);
  
  describe('anyMethodToBeTested', () => {
    it('should do something successfully', () => {
      // put the actual and expectation here 
    });
  });
})
```

### Steps to run the tests
1. Open a terminal at root project directory.
2. Run test command, `npm run test:integration`. Note that we don't need to call pretest and posttest commands. NestJS will call pretest before the test, and call posttest after all test cases are passed.   
3. Npm will first run pretest script.
4. Wait the tests until it stops.
5. The test is done.
- if tests are passed, npm will run posttest script.
- else, test will be stopped without running the posttest. It allows us to fix the bug and then rerun the test command.
