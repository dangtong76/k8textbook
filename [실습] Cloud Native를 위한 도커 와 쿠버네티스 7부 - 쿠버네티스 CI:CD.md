# Cloud Native를 위한 도커 와 쿠버네티스 7부 -CI/CD

## Node 및 NPM 설치

###  Mac

- node 설치

```{zsh}
brew update
brew install node
```

- 설치 확인

```{zsh}
node -v
npm -v
```

- 업그레이드

```{zsh}
brew upgrade node
```

- 삭제

```{zsh}
brew uninstall node
```

###  Windows

- node / npm 설치 : https://nodejs.org/ko/download/

![image-20210610190515021](/Users/dangtongbyun/Dropbox/05.Lecture/01.Kubernetes/reactWithNodes/img/image-20210610190515021.png)

## Project Setup



###  React Project Setup

```{bash}
# 수행하기 전에 vscode restart 해야함
mkdir cloudnative
cd cloudnative
npx create-react-app frontend
npm install axios

# 테스트 
npm start

```



Bootstrap 사용을 위해 Public 디렉토리 밑에 index.html 에 다음 내용을 Copy

```bash
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
```



###   Node.js(Typescript) Project Setup

```{bash}
# cloudNative 디렉토리 밑에서 ....
mkdir getmem
mkdir nodecheck
mkdir sayservice

cd getmem
npm init -y
npm install typescript -g
npm install typescript ts-node-dev express @types/express cors @types/cors
# vscode restart 해야함
tsc --init

cd nodecheck
npm init -y
npm install typescript ts-node-dev express @types/express cors @types/cors
tsc --init

cd sayservice
npm init -y
npm install typescript ts-node-dev express @types/express cors @types/cors
tsc --init

```



## Source Code 작성

### getmem 서비스 

아래과 같이 폴더를 생성 합니다.

~~~{bash}
# cloudNative/getmem 디렉토리에서
mkdir src
mkdir -p src/module
~~~

- src/Index.ts 작성

  ```{typescript}
  import express from 'express';
  import cors from 'cors';
  import { getmemRouter } from './module/getmem';
  
  const app = express();
  
  app.use(express.json());
  app.use(cors());
  app.use(getmemRouter);
  
  app.all('*', async (req, res) => {
    res.send({});
  });
  
  app.listen(3001, () => {
    console.log('getcpu app started. listen on 3001 port.');
  })
  ```

- src/module/getmem.ts 작성

  ```{typescript}
  import express, { Request, Response }from 'express';
  import os from 'os';
  
  const router = express.Router();
  const meminfo = os.totalmem();
  router.get('/api/getmem', (req: Request, res: Response) => {
    console.log(meminfo);
    res.send({meminfo});
  });
  
  export { router as getmemRouter }
  ```

- Package.json scripts 부분 수정

  ```{bash}
  "start": "ts-node-dev src/index.ts"
  ```

  

### nodecheck 서비스

```{bash}
# cloudNative/nodecheck 디렉토리에서
mkdir src
mkdir -p src/module
```

- Src/index.ts  작성

  ```{typescript}
  import express from 'express';
  import cors from 'cors';
  import { checkstatusRouter } from './module/nodecheck';
  
  const app = express();
  
  app.use(express.json());
  app.use(cors());
  
  app.use(checkstatusRouter);
  
  app.all('*', async (req, res) => {
    res.send({});
  });
  
  app.listen(3002, () => {
    console.log('checkstatus app started. listen on 3002 port.');
  })
  ```

- src/module/nodecheck.ts 작성

  ```{typescript}
  import express, { Request, Response }from 'express';
  import os from 'os';
  
  const router = express.Router();
  const message = "this is app1. you've hit " + os.hostname() + "\n";
  
  router.get('/api/nodecheck', (req: Request, res: Response) => {
    console.log(message);
    res.send({message});
  });
  
  export { router as checkstatusRouter }
  ```

- Package.json scripts 부분 수정

  ```{bash}
  "start": "ts-node-dev src/index.ts"
  ```

### sayservice 작성

```{bash}
# cloudNative/sayservice 디렉토리에서
mkdir src
mkdir -p src/module
```

- src/index.ts 작성

  ```{typescript}
  import express from 'express';
  import cors from 'cors';
  
  import { sayhelloRouter } from './module/sayservice';
  
  const app = express();
  
  app.use(express.json());
  app.use(cors());
  
  app.use(sayhelloRouter);
  
  app.all('*', async (req, res) => {
    res.send({});
  });
  
  app.listen(3003, () => {
    console.log('sayhello app started. listen on 3003 port.');
  })
  ```

- src/module/sayservice.ts 작성

  ```{typescript}
  import express, { Request, Response }from 'express';
  import os from 'os';
  
  const router = express.Router();
  const message = "hi my name is sayhello application \n";
  router.get('/api/sayservice', (req: Request, res: Response) => {
    console.log(message);
    res.send({message});
  });
  
  export { router as sayhelloRouter }
  ```

- Package.json scripts 부분 수정

  ```{bash}
  "start": "ts-node-dev src/index.ts"
  ```


### frontend  서비스 작성 (React)

#### src 밑의 모든 파일 삭제

Template 으로 들어 있는 모든 파일을 삭제 합니다.

#### src/Index.js

```{bash}
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

#### src/GetMem.js

```{react}
import React, { useState, useEffect } from 'react';
import axios from 'axios';


export function GetMem ()  {
  const [memInfo, setMemInfo] = useState([]);

  const getmemUrl = process.env.REACT_APP_GETMEM_URL || 'http://localhost:3001/api/getmem';
  
  const fetchMemInfo = async () => {
    const res = await axios.get(getmemUrl);
    setMemInfo(res.data);
  };

  useEffect(() => {
    fetchMemInfo();
  }, []);

  const renderedMemInfo = Object.values(memInfo).map((mem,i) => {
    console.log(mem);
    return (
    <div
      className="card" 
      style={{ width: '30%', marginBottom: '20px'}}
      key={i}
    >
      <div className='card-body'>
        <h3>{mem / 1024 / 1024 /1024} GB</h3>
      </div>
    </div>    
    );
    });
  
  return (
    <div className="d-flex flex-row flex-wrap justify-content-between">
      {renderedMemInfo}
    </div>
  );

};
```

#### src/NodeCheck.js

```{react}
import React, { useState, useEffect } from 'react';
import axios from 'axios';


export function NodeCheck ()  {
  const [nodeInfo, setNodeInfo] = useState([]);

  const nodecheckUrl = process.env.REACT_APP_NODECHECK_URL || 'http://localhost:3002/api/nodecheck';
  console.log('REACT_APP NodeCheckURL: ' + process.env.REACT_APP_NODECHECK_URL);
  const fetchNodeInfo = async () => {
    const res = await axios.get(nodecheckUrl);
    setNodeInfo(res.data);
  };

  useEffect(() => {
    fetchNodeInfo();
  }, []);
  
  const renderedNodeInfo = Object.values(nodeInfo).map((node,i) => {
    console.log(node);
    return (
    <div
      className="card" 
      style={{ width: '30%', marginBottom: '20px'}}
      key={i}
    >
      <div className='card-body'>
        <h3>{node}</h3>
      </div>
    </div>    
    );
    });


  return (
    <div className="d-flex flex-row flex-wrap justify-content-between">
      {renderedNodeInfo}
    </div>
  );

};
```

#### src/SayService.js

```{react}
import React, { useState, useEffect } from 'react';
import axios from 'axios';


export function SayService ()  {
  const [sayInfo, setSayInfo] = useState([]);

  const sayserviceUrl = process.env.REACT_APP_SAYSERVICE_URL || 'http://localhost:3003/api/sayservice';

  const fetchSayInfo = async () => {
    const res = await axios.get(sayserviceUrl);
    setSayInfo(res.data);
  };

  useEffect(() => {
    fetchSayInfo();
  }, []);
  
  const renderedSayInfo = Object.values(sayInfo).map((say,i) => {
    console.log(say);
    return (
    <div
      className="card" 
      style={{ width: '30%', marginBottom: '20px'}}
      key={i}
    >
      <div className='card-body'>
        <h3>{say}</h3>
      </div>
    </div>    
    );
    });

  return (
    <div className="d-flex flex-row flex-wrap justify-content-between">
      {renderedSayInfo}
    </div>
  );

};
```

#### src/App.js 

```{react}
import React from 'react';
import { GetMem } from './GetMem';
import { NodeCheck } from './NodeCheck';
import { SayService } from './SayService';


const App = () => {
  return <div>
    <h1>MemoryInfo</h1>
    <GetMem />
    <h1>node status</h1>
    <NodeCheck />
    <h1>Say Service</h1>
    <SayService />
  </div>;
};

export default App;
```



## Dockerfile Setup

### getmem

- Dockerfile

```dockerfile
FROM node:alpine

WORKDIR /app
COPY package.json .
RUN npm install
COPY . .

CMD ["npm", "start"]
```

- .dockerignore 파일을 아래와 같이 작성

```{docker}
node_modules
```

- .gitignore

```{bash}
node_modules
```



### nodecheck

- Dockerfile

```{dockerfile}
FROM node:alpine

WORKDIR /app
COPY package.json .
RUN npm install
COPY . .

CMD ["npm", "start"]
```

- .dockerignore 파일을 아래와 같이 작성

```{docker}
node_modules
```

- .gitignore

```{bash}
node_modules
```



### sayservice

- Dockerfile 작성

```dockerfile
FROM node:alpine

WORKDIR /app
COPY package.json .
RUN npm install
COPY . .

CMD ["npm", "start"]
```

- .dockerignore 파일을 아래와 같이 작성

```{docker}
node_modules
```

- .gitignore

```{bash}
node_modules
```



### frontend

-  Dockerfile  작성

```{bash}
FROM node:alpine

WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json .
COPY package-lock.json .
RUN npm install --silent
COPY . .

CMD ["npm","start","dev"]
```

- .dockerignore 작성

```{bash}
node_modules
build
```

- .gitignore

```{bash}
node_modules
build
```



## Kaffold Setup



### Manifest 침조 링크

https://skaffold.dev/docs/references/yaml/#deploy-helm

### MAC(setup)

```{zsh}
# Brew install
brew install skaffold

# For macOS on AMD64
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-darwin-amd64 && \
sudo install skaffold /usr/local/bin/

# For macOS on ARM64
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-darwin-arm64 && \
sudo install skaffold /usr/local/bin/

# MacPorts install
sudo port install skaffold
```

### Windows(setup)

다운로드 링크 : https://storage.googleapis.com/skaffold/releases/latest/skaffold-windows-amd64.exe

>  다운로드 후에 시스템 PATH 환경변수에  scaffold.exe 로 이름을 변경해서 파일의 경로를 넣어 주어야 함.
>
>  예) C:\Users\chungsju\skaffold

### skaffold yaml 파일 작성

> https://skaffold.dev/docs/references/yaml/ 

```{yaml}
apiVersion: skaffold/v2alpha3
kind: Config
# 쿠버네티스 배포 설정
deploy:
  kubectl:
    manifests:
      - ./xinfra/kubernetes/*
    defaultNamespace: devel
# 도커 이미지 빌드 관련 설정
build:
  local:
    push: true
  artifacts:
    - image: dangtong76/getmem
      context: getmem
      docker:
        dockerfile: Dockerfile
    - image: dangtong76/nodecheck
      context: nodecheck
      docker:
        dockerfile: Dockerfile
    - image: dangtong76/sayservice
      context: sayservice
      docker:
        dockerfile: Dockerfile
    - image: dangtong76/frontend
      context: frontend
      docker:
        dockerfile: Dockerfile
      # 소스 싱크 관련 설정
      sync:
        manual:
          - src: 'src/**/*.ts'
            dest: .
```

> https://skaffold.dev/docs/references/yaml/#deploy-helm 참조

###  skaffold 수행

#### skaffold 수행 방법

| 명령어         | 설명                                                         |
| -------------- | ------------------------------------------------------------ |
| skaffold run   | 도커 이미지 Build 후 레지스티리에 Push 한 뒤,  Deploy 까지 1회만 수행 |
| skaffold dev   | 도커 이미지 Build 후 레지스티리에 Push 한 뒤,  Deploy 까지 지속적으로 수행 |
| skaffold debug | skaffold dev 를 Debug 모드로 수행                            |

#### 네임스페이스 생성

```{bash}
kubectl create ns devel
```

#### skaffold 적용

```{bash}
skaffold dev
```



## Kubernetes Yaml 을 Git 리포지토리와 연동

###  Git 설정



### /etc/hosts 파일 변경

```{bash}
kubectl get ingress

# mac
sudo vi /etc/hosts

# windows
hosts 파일 수정
```





### 2.3 git setup

```{bash}
git init
```

```{bash
build
node_modules



node_modules


xinfra
.DS_Store
.vscode
```

## 7. git se

