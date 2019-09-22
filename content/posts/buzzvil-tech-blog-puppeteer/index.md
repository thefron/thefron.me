---
title: "회사 테크 블로그 기고 - PhantomJS를 Headless Chrome(Puppeteer)로 전환하며"
date: 2019-09-22T11:49:39+09:00
tags:
  - docker
  - puppeteer
comments: true
draft: false
---

> 이 포스팅은 제가 [버즈빌 테크 블로그](https://www.buzzvil.com/ko/2019/01/08/1tech-blog-phantomjs를-headless-chromepuppeteer로-전환하며/)에 기고한 글을 전제한 것입니다.

버즈빌에서는 모바일 잠금화면에 내보내기 위한 광고 및 컨텐츠 이미지를 생성하기 위한 PhantomJS 렌더링 서버를 다수 운영하고 있습니다. 일반적으로 PhantomJS는 웹페이지 캡쳐에 많이 쓰이지만, 기본적으로 headless하게 웹페이지를 렌더링하고 캡쳐할 수 있다는 특성 때문에 동적인 이미지 생성에도 많이 활용됩니다. 버즈빌의 렌더링 서버는 200개 이상의 컨텐츠 프로바이더로부터 실시간으로 잠금화면 컨텐츠 이미지를 생성하고 있어 분당 수백 건의 이미지를 안정적으로 생성하는 것이 가능해야 합니다.
![버즈빌 게임](https://www.buzzvil.com/wp-content/uploads/2018/11/Buzzvil-Games-1-1.jpg) 렌더링 서버의 스케일링 이슈를 해결하기 위해 버즈빌에서는 여러 대의 렌더링 서버를 둬서 횡적으로 확장을 함과 동시에, 개별 서버 내에서도 리소스 사용률을 높이기 위해 [Ghost Town](https://github.com/Buzzvil/ghost-town)이라는 라이브러리를 작성해 PhantomJS 프로세스 풀을 구성하여 사용하고 있었습니다([Scaling PhantomJS With Ghost Town](https://www.www.buzzvil.com/2014/05/29/scaling-phantomjs-ghost-town/) )
한편, 시간이 지나면서 잠금화면에서 렌더링하는 이미지 템플릿의 종류가 다양해지고, emoji 및 여러 특수문자를 표현하기 위해 렌더링 서버에 여러 폰트(대표적으로 Noto Sans CJK)를 설치해야 하는 요구사항이 추가됐는데, PhantomJS에서 폰트 렌더링이 일관적이지 않은 문제가 발생했습니다. ![동일한 템플릿이지만 폰트가 비일관적으로 렌더링되고 있는 모습](https://www.buzzvil.com/wp-content/uploads/2019/01/Screen-Shot-2019-01-08-at-11.46.47-AM.png) 이 문제의 정확한 원인은 결국 찾지 못했지만 PhantomJS의 이슈였거나 시스템 상에 폰트가 시간이 지나면서 추가 설치됨에 따라 font cache가 서버마다 일관되지 않은 상태가 되었기 때문인 것으로 짐작하고 있습니다. 다른 워크로드와 마찬가지로 렌더링 서버도 최초에는 packer를 이용해 일관되게 이미지를 빌드하고 업데이트하려고 했지만, 자주 기능이 추가되거나 배포되는 서비스가 아니기에 서버를 오래 띄워놓고 수동으로 유지보수를 한 케이스들이 누적되어 더 이상 packer를 이용해 시스템이나 폰트를 최신 상태로 유지하는 것이 어려운 상태였습니다. 모든 눈꽃송이가 자세히 보면 조금씩 다르게 생겼다는 것에서 비롯된 [snowflake](https://martinfowler.com/bliki/SnowflakeServer.html), 즉 배포된 서버들이 시간이 지남에 따라 조금씩 다른 상태가 된 것입니다. 평소에는 문제가 없어 보이지만, 추가적인 확장성이 필요해 scale out을 하거나 새로운 템플릿을 개발해 배포를 하면 문제가 발생하는 상황이었습니다.
사실 더 큰 문제는 PhantomJS 프로젝트가 더 이상 관리되지 않는다는 점이었습니다. 2017년 Google Chrome 59버전부터 Headless Chrome이 내장되기 시작하였고, 곧바로 Node API인 puppeteer가 릴리즈 되어, 현시점에서 가장 많이 쓰이는 렌더링 엔진을 손쉽게 headless로 사용할 수 있는 환경이 되었습니다. 때문에 PhantomJS 관리자가 사실상의 [중단을 선언](https://groups.google.com/forum/#!topic/phantomjs/9aI5d-LDuNE)하였고, 2018년에는 최초 개발자에 의해 프로젝트가 [아카이브 되었습니다](https://github.com/ariya/phantomjs/issues/15344). 프로젝트가 업데이트되지 않는 것은 템플릿에 최신 CSS 스펙을 사용하지 못한다는 것을 의미하고, 버그 수정도 되지 않기에 어플리케이션의 유지보수가 굉장히 어려워짐을 의미합니다. 현재까지의 문제점을 정리하면 아래와 같습니다.

1. 자주 배포되지 않는 서비스 특성으로 인한 서버들이 snowflake화 되는 현상(특히 폰트)
2. PhantomJS의 개발 중단으로 인해 버그 픽스 및 최신 CSS 속성 사용이 어렵게 되고, 향후 유지보수나 새로운 템플릿 개발이 어려워짐

해결방안은 명확했습니다. 첫번째 문제를 해결하기 위해서는 어플리케이션과 폰트가 설치된 시스템을 통째로 컨테이너로 만들고, CI/CD 파이프라인을 통해 지속적으로 빌드하여 snowflake화 되지 않도록 하면 됩니다. 사실 최초에 packer를 이용해 AMI 이미지를 생성하도록 구성이 되어있었기에, 매 배포마다 AMI를 새로 생성하고 지속적으로 렌더링 서버를 배포하는 환경이기만 했으면 snowflake를 방지할 수 있었을 것입니다. 하지만 자주 기능이 추가되거나 배포되는 서비스가 아닌데다, AMI를 빌드하는 과정이 CI/CD에 통합돼 있지 않고 어플리케이션만 지속적으로 배포하는 환경이었기에 편의상 서버를 종료하지 않고 장기간 관리를 해 오게 되었고, packer로 새로운 AMI 이미지를 빌드하는 것이 어려워 졌습니다. 때문에 AMI 빌드를 통한 배포 대신, 이미 운영 중인 kubernetes 클러스터에 도커 컨테이너를 빌드해 immutable한 형상으로 배포하기로 결정하였습니다.
두번째 문제의 간단한 해결책은 PhantomJS를 puppeteer로 변경하는 것입니다. 이 부분은 생각보다 간단했습니다. 의도했는지는 알 수 없으나 puppeteer의 api는 PhantomJS와 꽤나 비슷합니다. drop-in replacement까진 아니지만, PhantomJS api 호출하는 부분만 살짝 바꿔주는 정도로 교체가 가능하였습니다. 물론 교체만 하였다고 해서 기존에 개발된 템플릿이 의도된 대로 출력되는 것을 보장하지는 않기에, 렌더링 서버가 렌더링하는 수많은 템플릿들을 PhantomJS와 puppeteer로 각각 출력하여 일일히 비교하는 작업이 필요했습니다.
어떤 템플릿이 어떤 인자를 필요로하며 의도된 출력 결과가 무엇인지에 대한 정의가 남아있지 않았기에 템플릿마다 샘플 케이스들을 생성하는 작업이 필요했습니다. 아직까지는 수동으로 결과를 비교해야하는 문제점이 있지만 적어도 직접 확인할 수 있는 것은 큰 도움이 되었습니다. 향후에는 자동화된 테스트 케이스를 구성하여 기능 개발이 좀 더 용이하도록 보완할 계획입니다. 결과는 만족스러웠습니다. 많은 경우 기존과 출력 결과가 달랐지만, 최신의 크롬 웹킷이 사용되면서 오히려 템플릿을 개발할 때 의도했던대로 CSS를 더 정확하게 렌더링하게 된 것이었습니다.

```Dockerfile
FROM node:10-slim

RUN apt-get update && \
apt-get install -yq gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 \
libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 \
libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 \
libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 \
fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst ttf-freefont \
ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils wget unzip && \
wget https://github.com/Yelp/dumb-init/releases/download/v1.2.1/dumb-init_1.2.1_amd64.deb && \
dpkg -i dumb-init_*.deb && rm -f dumb-init_*.deb && \
apt-get clean && apt-get autoremove -y && rm -rf /var/lib/apt/lists/*

RUN yarn global add puppeteer@1.10.0 && yarn cache clean

ENV NODE_PATH="/usr/local/share/.config/yarn/global/node_modules:${NODE_PATH}"

RUN groupadd -r pptruser && useradd -r -g pptruser -G audio,video pptruser

# Set language to UTF8
ENV LANG="C.UTF-8"

RUN wget -P ~/fonttmp \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoSans-unhinted.zip \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoSansCJKjp-hinted.zip \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoSansCJKkr-hinted.zip \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoSansCJKtc-hinted.zip \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoSansCJKsc-hinted.zip \
    https://noto-website-2.storage.googleapis.com/pkgs/NotoColorEmoji-unhinted.zip \
 && cd ~/fonttmp \
 && unzip -o '*.zip' \
 && mv *.*tf /usr/share/fonts \
 && cd ~/ \
 && rm -rf ~/fonttmp

WORKDIR /app
# Add user so we don't need --no-sandbox.
RUN mkdir /screenshots && \
    mkdir -p /home/pptruser/Downloads && \
    mkdir -p /app/node_modules && \
    chown -R pptruser:pptruser /home/pptruser && \
    chown -R pptruser:pptruser /usr/local/share/.config/yarn/global/node_modules && \
    chown -R pptruser:pptruser /screenshots && \
    chown -R pptruser:pptruser /usr/share/fonts && \
    chown -R pptruser:pptruser /app

# Run everything after as non-privileged user.
USER pptruser

RUN fc-cache -f -v

COPY --chown=pptruser:pptruser package*.json /app/
RUN npm install && \
    npm cache clean --force
COPY --chown=pptruser:pptruser . /app/

ENTRYPOINT ["dumb-init", "--"]
CMD ["npm", "start"]
```

puppeteer를 사용하면서 약간의 권한 문제가 있어서 결과적으로 위와 같은 Dockerfile을 작성하게 되었는데, puppeteer 도커 이미지 작성에 관한 최신 정보는 [여기](https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md)서 확인할 수 있습니다. 컨테이너 오케스트레이션(K8s)을 사용하면 process 기반의 스케일링은 컨테이너를 여러대 띄워 로드밸런싱을 손쉽게 할 수 있지만, 개별 컨테이너의 throughput을 향상시키기 위해 기존에 Ghost town을 작성해 PhantomJS 프로세스 풀을 만든 것처럼 크롬 프로세스 풀을 구성하기로 하였습니다. 프로세스 풀 구성에는 [generic-pool](https://github.com/coopernurse/node-pool) 라이브러리를 사용하였으며 아래처럼 구성하였습니다.

```javascript
const puppeteer = require("puppeteer");
const genericPool = require("generic-pool");

const puppeteerArgs = ["--no-sandbox", "--disable-setuid-sandbox", "--disable-dev-shm-usage"];

const createPuppeteerPool = ({
  max = 5,
  min = 2,
  maxUses = 50,
  initialUseCountRand = 5,
  testOnBorrow = true,
  validator = () => Promise.resolve(true),
  idleTimeoutMillis = 30000,
  ...otherConfig
} = {}) => {
  const factory = {
    create: async () => {
      const browser = await puppeteer.launch({ headless: true, args: puppeteerArgs });
      browser.useCount = parseInt(Math.random() * initialUseCountRand);
      return browser;
    },
    destroy: (browser) => {
      browser.close();
    },
    validate: (browser) => {
      return validator(browser)
        .then(valid => Promise.resolve(valid && (maxUses <= 0 || browser.useCount < maxUses)));
    }
  };

  const pool = genericPool.createPool(factory, { max, min, testOnBorrow, idleTimeoutMillis, ...otherConfig });
  const genericAcquire = pool.acquire.bind(pool);
  pool.acquire = () => genericAcquire().then(browser => {
    browser.useCount += 1;
    return browser;
  });
  pool.use = (fn) => {
    let resource;
    return pool.acquire()
      .then(r => {
        resource = r;
        return resource;
      })
      .then(fn)
      .then((result) => {
        pool.release(resource);
        return result;
      }, (err) => {
        pool.release(resource);
        throw err;
      });
  };
  return pool;
};

module.exports = createPuppeteerPool;
```

### Caveats

PhantomJS에서 puppeteer로 전환함에 있어서 몇가지 주의해야 할 점이 있었는데요. 첫째는 기존에 사용하던 템플릿의 html에 이미지 소스를 `file://` url 프로토콜을 이용해 로드하는 경우가 있었는데, PhantomJS에서는 정상적으로 로드가 되지만 Headless Chrome에서는 보안 정책으로 인해 로컬 파일을 로드할 수 없었습니다([관련 이슈](https://github.com/GoogleChrome/puppeteer/issues/1643)). 때문에 로컬 이미지가 필요한 템플릿은 Express 서버에서 static file serving을 하도록 하고 `http://` 프로토콜로 변경하였습니다. 다음으로 발생한 문제는 PhantomJS을 이용한 기존 구현에서는 jade template을 compile한 후 page 객체의 `setContent` 메소드를 이용해 html을 로드하였는데, puppeteer에서는 `page#setContent` API 호출 시 외부 이미지가 로드될 때까지 기다리지 않는다는 점입니다. [puppeteer 에 올라온 관련 이슈](https://github.com/GoogleChrome/puppeteer/issues/728)에서는 `setContent` 대신 아래와 같이 html content를 data URI로 표현하고 `page#goto`의 인자로 넘기면서 `waitUntil` 옵션을 주는 방식을 해결방법으로 권하고 있습니다. 이 때 주의해야 할 점은 `waitUntil`의 옵션으로 `networkidle0`이나 `networkidle2` 등을 사용하면 외부 이미지가 충분히 로드될 때 까지 기다리는 것은 맞지만, 500ms 이내에 추가적인 네트워크 커넥션이 발생하지 않을 때까지 기다리는 옵션이기 때문에 외부 이미지가 로드되더라도 추가적으로 500ms를 기다리게 됩니다. 때문에 SPA 웹페이지를 캡쳐하는 경우가 아니라 정적인 html을 로드하는 경우라면 `load` 이벤트로 지정하면 됩니다. 이외에도 향후에 프로젝트의 유지관리나 운영 중인 서비스의 모니터링을 위해 Metrics API 엔드포인트를 만들어 prometheus에서 메트릭을 수집할 수 있도록 하고 grafana 대시보드를 구성하였습니다. 이 대시보드는 어떤 템플릿이 실제로 사용되고 있는지, 템플릿 렌더링에 시간이 얼마나 소요되는지 등을 모니터링할 수 있도록 구성하여 사용되지 않고 있는 템플릿을 판단하거나 서비스 지표를 모니터링 하는 데 이용하고 있습니다. ![grafana와 prometheus를 이용해 구현한 렌더링 서버 모니터링 대시보드.](https://www.buzzvil.com/wp-content/uploads/2019/01/Screen-Shot-2019-01-08-at-11.52.15-AM.png)

### 마치며

최근에 들어서는 PhantomJS를 사용하던 많은 곳에서 puppeteer로의 전환을 해오고 있어 본 포스팅에서 다루고 있는 내용이 크게 새로운 내용은 아닐 수 있습니다. 하지만 버즈빌에서는 렌더링 서버가 과거에 이미 PhantomJS를 사용하는 것을 전제로 상당한 최적화가 진행되어 왔고, 꽤나 높은 동시 처리량이 요구되는 상황에서 puppeteer로 교체를 해버리기에는 여러 불확실한 요소들이 존재하는 상황이었습니다. 버즈빌의 핵심 비즈니스 중 하나인 잠금화면에 사용되는 이미지를 렌더링하는 서비스가 레거시(개발이 중단된 PhantomJS)에 의존하는 코드베이스 때문에 변경이 어려워지는 것은 향후 꽤나 큰 기술부채로 작용할 것이라 판단하였습니다. 이번 마이그레이션을 진행하면서는 이 부분을 염두에 두고 컨테이너를 사용해 CI/CD 파이프라인을 구축해 지속적으로 컨테이너 기반의 이미지를 생성하도록 변경하였고, 그 결과는 꽤나 만족스러웠습니다. 마이그레이션 이후 그간 밀려 있던 신규 템플릿 개발이나 신규 컨텐츠 프로바이더를 추가하는 과정이 수월해졌기 때문입니다. 빠르게 변화하는 비즈니스 요구사항에 대응하다보면 기술부채는 필연적으로 쌓일 수밖에 없습니다. 개발자에게는 당연히 눈에 보이는 모든 기술부채들을 청산하고 싶은 욕구가 있지만 늘 빚 갚는데 시간을 쓰고 있을 수만은 없는 노릇입니다. 리소스에는 한계가 있으니까요. 어떤 기술부채를 지금 당장 해결해야하는지 의사결정을 하는데 있어 고민이 된다면 일단 “측정”을 해보는 것을 권장합니다. 수치화된 지표가 있다면 당장 의사결정권자나 팀을 설득하는 데 사용할 수도 있지만, 서비스의 핵심 지표들을 하나 둘씩 모니터링 해나가다 보면 서비스에 대한 가시성이 높아지고 미래에 정말로 병목이 되는 지점을 찾아내기 쉬워질 것입니다.

### 참고 자료

* [https://docs.browserless.io/blog/2018/06/04/puppeteer-best-practices.html](https://docs.browserless.io/blog/2018/06/04/puppeteer-best-practices.html)
* [https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md)
* Icons made by [Freepik](https://www.freepik.com/) from [Flaticon](https://www.flaticon.com/) is licensed by [Creative Commons](http://creativecommons.org/licenses/by/3.0/) BY 3.0
