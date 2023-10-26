# Optimize Dockerfile for BuildKit and Layer Caching

## Dockerfile before optimize
```dockerfile
FROM node:20 as build

WORKDIR /usr/src/app

COPY . .

RUN npm ci

RUN npm run build

RUN npx prisma generate

FROM node:20

WORKDIR /usr/src/app

RUN apt update \
    && apt install -y dumb-init

# A wildcard is used to ensure both package.json AND package-lock.json are copied
COPY package*.json ./

RUN npm ci --omit=dev

COPY --from=build /usr/src/app/dist/ ./dist/
COPY --from=build /usr/src/app/config/ ./config/
COPY --from=build /usr/src/app/node_modules/.prisma/ ./node_modules/.prisma/

ENV APP_ENV development


EXPOSE 8080

ENTRYPOINT ["/usr/bin/dumb-init", "--"]

CMD ["node", "dist/src/main.js"]
```


## Dockerfile after optimize
```dockerfile
# Build stage
FROM node:20 as build

WORKDIR /usr/src/app

# Copy package files first to leverage Docker build cache (caches node modules unless these files change)
COPY package.json package-lock.json ./

# Using BuildKit's caching mechanism to cache the npm dependencies
# if package.json and package-lock.json have not changed,
# This speeds up the build process by reusing previously installed node_modules
RUN --mount=type=cache,target=/root/.npm npm ci

# Copy the rest of the files
COPY . .

RUN npx prisma generate

RUN npm run build

# Remove devDependencies in node_modules
RUN npm prune --production

# Runtime stage
FROM node:20

WORKDIR /usr/src/app

# Install dumb-init
RUN apt update && apt install -y dumb-init

# Copy necessary files from build stage for runtime
COPY --from=build /usr/src/app/dist/ ./dist/
COPY --from=build /usr/src/app/config/ ./config/
COPY --from=build /usr/src/app/node_modules/ ./node_modules/

ENV APP_ENV development

EXPOSE 8080

ENTRYPOINT ["/usr/bin/dumb-init", "--"]

CMD ["node", "dist/src/main.js"]
```

## 1. --mount 옵션을 줘도 ECS service가 업데이트 될 때 container 자체가 전부 바뀌는데 의미가 있는가?

Docker의 **`--mount=type=cache`** 옵션은 주로 로컬 머신이나 CI/CD 파이프라인과 같은 지속적인 빌드 환경에서 빌드 시간을 단축하기 위해 사용됩니다. 캐시는 해당 빌드 환경에서만 유효하며, 이전에 빌드한 결과를 다음 빌드에 활용하기 위해 사용됩니다.

Amazon ECS의 경우, 새로운 버전의 서비스를 배포할 때마다 새로운 Docker 이미지를 생성하고 해당 이미지를 기반으로 새로운 컨테이너 인스턴스를 시작합니다. 이 때문에, 캐시가 ECS의 각 인스턴스에서 효과적으로 활용되지 않을 수 있습니다.

그러나, ECS 배포 전에 CI/CD 파이프라인에서 이미지를 빌드하는 경우가 많습니다. 이때, CI/CD 파이프라인 내에서 캐시를 활용하여 빌드 시간을 단축하는 것은 매우 유용합니다. CI/CD 파이프라인에서 빌드 속도가 빨라지면 전체 배포 시간도 줄어들게 됩니다.

결론적으로, ECS 서비스의 업데이트에서 직접적인 캐싱의 이점을 얻기는 어렵지만, CI/CD 파이프라인에서 Docker 이미지를 빌드하는 과정에서 캐시를 활용하면 전체적인 배포 시간을 단축시킬 수 있습니다.

---

## 2. package.lock, package.json 만 먼저 복사해와서 의존성을 설치한 이후에 나머지 파일들을 복사하는 게 Optimizing builds with cache management 에 해당되는가?

Docker 이미지를 빌드할 때, **`Dockerfile`**의 각 명령은 별도의 레이어를 생성합니다. 이러한 레이어는 캐시될 수 있으며, 동일한 명령이 다음 빌드에서 실행될 때 이전 빌드의 결과를 재사용할 수 있습니다.

이러한 캐싱 메커니즘을 최대한 활용하기 위해서는 빈번하게 변경되는 파일들의 복사를 나중에 수행하는 것이 좋습니다. 예를 들어, **`package.json`** 및 **`package-lock.json`**은 프로젝트의 의존성이 변경되지 않는 한 자주 바뀌지 않습니다. 따라서 이 두 파일을 먼저 복사하고 **`npm ci`** 명령을 실행하면 해당 레이어는 캐시될 확률이 높아집니다.

반면, 소스 코드와 같은 나머지 파일들은 개발 과정에서 빈번하게 변경될 수 있습니다. 이러한 파일들은 나중에 복사하여, 이전 단계에서 생성된 캐시 레이어가 무효화되지 않도록 합니다.

즉, 위 **`Dockerfile`**에서 **`package.json`** 및 **`package-lock.json`**을 먼저 복사하고 의존성을 설치한 후에 나머지 파일들을 복사하는 접근 방식은 Docker 빌드의 캐시 관리를 최적화하는 데 도움이 됩니다.

---

## 3. Jenkinsfile에서 Dockerfile로 이미지를 빌드해서 ECR에 업로드, 새로운 task-definition으로 서비스를 업데이트하는데 캐시되면 어디에 저장되고 이전빌드의 결과를 재사용할 수 있는가?

Docker 빌드 캐시는 도커 데몬 호스트에 저장됩니다. **`Dockerfile`**의 각 단계를 실행할 때마다 해당 단계의 결과는 로컬 시스템의 Docker 레이어로 저장되며, 동일한 명령이 다음 번에 실행될 때 이전에 생성된 레이어를 재사용합니다.

하지만 Jenkins와 같은 CI/CD 파이프라인에서는 일반적으로 매번 새로운 환경에서 빌드를 실행하기 때문에 Docker 빌드 캐시를 효과적으로 활용하기가 어려울 수 있습니다. 여기서 중요한 점은, Docker 빌드 캐시는 빌드 환경 내에서만 유효하다는 것입니다. 따라서 새로운 빌드 환경에서는 이전 빌드의 결과를 재사용할 수 없습니다.

그렇기 때문에 CI/CD 환경에서 Docker 빌드 캐시를 효과적으로 활용하려면 몇 가지 추가적인 전략이 필요합니다:

1. **영구적인 빌드 환경**: Jenkins 슬레이브나 노드를 고정하고, 이 노드에서만 Docker 이미지 빌드를 수행하여 빌드 캐시를 유지합니다.
2. **외부 캐시 저장소 활용**: BuildKit의 캐싱 기능과 외부 저장소(예: S3)를 사용하여 빌드 캐시를 저장하고 다시 가져옵니다. 이 방법을 사용하면 여러 빌드 노드 간에 캐시를 공유할 수 있습니다.
3. **레이어 최적화**: **`Dockerfile`**을 작성할 때 변경이 빈번한 명령은 마지막에 배치하여 캐시 무효화를 최소화합니다.

위와 같은 전략을 사용하면 CI/CD 환경에서도 Docker 빌드 캐시를 효과적으로 활용할 수 있습니다. 그러나, 빌드 환경이나 워크플로에 따라 이러한 전략을 적용하는 것이 항상 적절한 것은 아닙니다. 따라서 현재의 파이프라인 구조와 요구사항을 고려하여 최적의 전략을 선택해야 합니다.

---

## 4. Dockerfile 의 build stage에서 RUN --mount=type=cache,target=/root/.npm npm ci 와 RUN npm ci 중 후자가 더 느린 이유

두 Dockerfile 사이의 주요 차이점은 **`npm ci`** 실행에 **`--mount=type=cache,target=/root/.npm`** 옵션이 있는지 여부입니다. 첫 번째 Dockerfile에서는 BuildKit의 캐시 메커니즘이 사용되고, 두 번째 Dockerfile에서는 사용되지 않습니다.

두 번째 Dockerfile에서 **`npm ci`** 명령을 실행할 때 더 많은 시간이 소요되는 이유는 다음과 같습니다:

1. **캐시 미사용**: **`-mount=type=cache`** 옵션이 없기 때문에 **`npm ci`** 명령이 실행될 때마다 모든 종속성을 새로 다운로드합니다. 첫 번째 Dockerfile에서는 이전에 다운로드한 종속성을 캐시에서 재사용할 수 있습니다.
2. **네트워크 지연**: 종속성들이 매번 다운로드되므로 네트워크 지연이 발생할 수 있습니다. 특히 많은 패키지가 있거나 대용량의 패키지가 있을 경우에는 더욱 시간이 많이 소요될 수 있습니다.
3. **디스크 I/O**: 종속성을 다운로드하고 압축을 풀기 위한 추가적인 디스크 작업이 필요합니다. 디스크 I/O는 CPU나 메모리 연산보다 더 많은 시간이 소요되므로 빌드 시간에 영향을 줄 수 있습니다.

따라서 BuildKit의 캐시 기능을 활용하지 않는 두 번째 Dockerfile의 경우, 매번 새로운 종속성을 다운로드하고 설치하는 작업 때문에 빌드 시간이 첫 번째 Dockerfile보다 더 길어집니다.

---

## 5. BuildKit의 캐시 메커니즘에 대해서 ..

BuildKit은 Docker 이미지 빌드 프로세스를 최적화하고, 동시성과 캐시 메커니즘을 향상시키기 위한 도구입니다. Docker 18.09 버전 이후부터는 **`DOCKER_BUILDKIT=1`** 환경 변수를 설정하면 Docker CLI에서 사용할 수 있습니다.

BuildKit의 주요 특징 중 하나는 고급 캐싱 메커니즘입니다. 기존 Docker 빌드는 각 단계마다 결과를 캐시하며, Dockerfile의 일부가 변경되면 해당 단계 이후의 모든 단계에서 캐시를 재사용하지 않습니다. BuildKit은 이런 접근법을 개선하여 더 세밀한 종속성 추적을 수행합니다.

BuildKit의 주요 캐싱 특징은 다음과 같습니다:

1. **레이어 기반의 캐싱**: 이전에는 Dockerfile의 각 명령어 단계별로 이미지 레이어가 캐싱되었습니다. BuildKit은 이를 최적화하여 실제로 변경된 내용만 빌드하도록 해줍니다.
2. **외부 캐시 소스**: BuildKit은 외부 소스 (예: S3, HTTP, local directory)에서 캐시를 가져오거나 저장할 수 있습니다.
3. **인라인 캐싱**: Dockerfile의 각 RUN 명령어에 **`-mount=type=cache`** 옵션을 사용하여 특정 경로를 캐시로 지정할 수 있습니다. 이렇게 하면 해당 경로에 있는 데이터는 빌드 간에 재사용됩니다. 예를 들어, Node.js 프로젝트에서 **`node_modules`** 디렉토리의 내용을 캐시하고 싶을 때 이 옵션을 사용할 수 있습니다.
4. **병렬성과 동시성**: BuildKit은 여러 단계의 동시 빌드를 지원하여 빌드 시간을 단축시킵니다. 변경되지 않은 단계는 재빌드 없이 캐시에서 가져올 수 있습니다.

이러한 기능 덕분에 BuildKit은 큰 프로젝트나 종속성이 많은 프로젝트에서 빌드 시간을 크게 단축시키는 데 효과적입니다. BuildKit의 캐싱 기능은 반복적인 빌드와 개발 중에 특히 유용하며, CI/CD 파이프라인에서도 매번 종속성을 다시 설치하지 않아도 되기 때문에 시간을 절약할 수 있습니다.

---

## 6. 근데 Jenkinsfile에서 Dockerfile로 이미지를 빌드해서 ECR에 업로드, 새로운 task-definition으로 서비스를 업데이트하는데 RUN --mount=type=cache,target=/root/.npm npm ci 이 명령어를 통해서 Jenkinsfile에서 docker buildx build --push --platform=linux/arm64,linux/amd64 -t ${ECR_REPOSITORY_URI}${API_IMAGE_REPO_NAME}:${env.API_IMAGE_TAG} -f infra/docker/app/Dockerfile . --progress plain이 stage가 실행될 때 arm, amd 각 두 운영체제로 빌드 될 때 두번째에는 캐싱된다는 의미인가?

**`RUN --mount=type=cache,target=/root/.npm npm ci`**를 사용하면, BuildKit은 해당 경로(**`/root/.npm`**)에 대한 캐싱을 수행하게 됩니다. 그럼으로써 동일한 **`package.json`**과 **`package-lock.json`**을 기반으로 하는 의존성 설치 작업에 대해 중복적인 작업을 수행하는 대신 캐시를 활용하여 더 빠른 빌드 시간을 달성할 수 있습니다.

그러나, 여기에서 주의할 점은 **`docker buildx build`**를 사용하면서 다양한 플랫폼(예: **`linux/arm64`** 및 **`linux/amd64`**)을 대상으로 빌드를 수행할 경우, 각 플랫폼은 독립적으로 빌드되며 그로 인해 각각의 플랫폼은 자신만의 캐시를 가질 수 있습니다.

답변을 요약하면:

- 첫 번째로 **`linux/arm64`** 플랫폼으로 빌드를 수행할 때, 의존성 설치 작업에 대한 캐시가 생성될 수 있습니다.
- 이후 **`linux/amd64`** 플랫폼으로 빌드를 수행할 때, 해당 플랫폼의 이전 빌드 결과(캐시)를 활용할 수 있습니다.

하지만 중요한 것은, Jenkins 빌드가 여러 번 수행되더라도 동일한 Jenkins 에이전트 또는 노드에서 실행되어야 캐시를 유지하고 재사용할 수 있다는 것입니다. 다른 에이전트나 노드에서 빌드를 수행한다면 해당 노드에는 처음부터 캐시가 없을 것이기 때문에 캐시를 재사용할 수 없습니다. 따라서 캐시의 효과를 극대화하기 위해서는 빌드 환경의 일관성을 유지하는 것이 중요합니다.

---

## 7. 동일한 Jenkins 에이전트 또는 노드 라는 말이 같은 Pipeline을 의미하는거야? 그리고 다른 아키텍쳐인데 같은 cache를 사용하면 호환이 안되는 문제에 대해서는 어떻게 되는 가?

1. **동일한 Jenkins 에이전트 또는 노드**:
    - Jenkins에는 'Master'와 'Agent' (또는 'Node')라는 두 가지 주요 구성 요소가 있습니다. Master는 작업을 관리하고, Agent (또는 Node)는 실제 작업을 수행합니다. 여기서 말하는 "동일한 Jenkins 에이전트 또는 노드"는 빌드 작업을 수행하는 실제 컴퓨터 또는 가상 머신을 의미합니다.
    - 이는 같은 Pipeline과는 다릅니다. 동일한 Pipeline 작업이 여러 번 수행되더라도 각 실행은 다른 에이전트 또는 노드에서 발생할 수 있습니다.
2. **다른 아키텍처에 대한 캐시 사용**:
    - 위에서 언급한 것처럼 각 아키텍처(예: **`linux/arm64`** 및 **`linux/amd64`**)는 독립적으로 빌드되기 때문에, 각각의 플랫폼은 자신만의 캐시를 가질 것입니다.
    - 따라서 **`linux/arm64`**의 빌드 작업이 **`linux/amd64`**의 캐시를 사용하거나 그 반대의 경우가 발생할 가능성은 없습니다. 각 아키텍처는 자신의 캐시를 생성하고 사용하므로 호환성 문제에 대해 걱정할 필요가 없습니다.

BuildKit은 다양한 아키텍처와 플랫폼에서의 빌드 작업을 효율적으로 관리하기 위해 설계되었으므로, 이러한 아키텍처 간의 캐시 교차 사용에 대한 문제는 발생하지 않습니다.

---

## 8. 현재 Jenkins에는 Built-in Node만 존재하는데 에이전트가 없는 상황인가?

Jenkins에서 "Built-In Node"는 Jenkins 서버 자체에서 실행되는 기본 노드를 의미합니다. 이것은 마스터 노드로도 불리며, 일반적으로는 작업을 직접 수행하기보다는 다른 에이전트 노드들을 관리하는 역할을 합니다.

추가 에이전트를 연결하지 않았다면, 현재 모든 Jenkins 작업은 이 "Built-In Node"에서 직접 실행되고 있을 것입니다. 따라서 에이전트는 따로 없는 상태입니다.

---

RUN --mount=type=cache,target=/root/.npm npm ci를 사용하면, BuildKit은 해당 경로(/root/.npm)에 대한 캐싱을 수행하게 됩니다. 그럼으로써 동일한 package.json과 package-lock.json을 기반으로 하는 의존성 설치 작업에 대해 중복적인 작업을 수행하는 대신 캐시를 활용하여 더 빠른 빌드 시간을 달성할 수 있다고 했는데, 이게 docker buildx와 관련이 있는거야?

네, 그렇습니다. **`--mount=type=cache`** 옵션은 Docker의 BuildKit을 사용할 때 사용되는 캐싱 메커니즘입니다. BuildKit은 Docker 18.09 버전부터 실험적으로 도입된 빌드 백엔드입니다. 이 캐시 메커니즘을 사용하려면 BuildKit을 활성화해야 합니다.

**`docker buildx`**는 BuildKit을 사용하는 명령어 확장입니다. 따라서 **`docker buildx`**를 사용하여 이미지를 빌드할 때는 BuildKit의 캐시 메커니즘이 활성화되어 있습니다. 이를 통해 **`--mount=type=cache`**와 같은 고급 캐싱 옵션을 사용할 수 있게 되며, 그 결과 빠른 빌드 시간을 달성할 수 있습니다.

---

## 정리 쪽 질문

<aside>
💡 정리하자면  jenkinsFile에서 

docker buildx build --push --platform=linux/arm64,linux/amd64 -t ${ECR_REPOSITORY_URI}${API_IMAGE_REPO_NAME}:${env.API_IMAGE_TAG} -f infra/docker/app/Dockerfile . --progress plain

커맨드가 있는 stage가 실행될 때 마다 나는 현재 build-in Node 에서만 실행되니까 캐싱이 된다는 이야기네?

</aside>

맞습니다. Jenkins에서 실행되는 해당 **`docker buildx build`** 커맨드는 BuildKit을 활용하므로 **`RUN --mount=type=cache`** 같은 캐싱 옵션을 사용할 수 있습니다. 그리고 현재 Jenkins 환경에서는 "Built-In Node"에서만 빌드가 실행되기 때문에 이전 빌드에서 생성된 캐시를 재사용할 수 있습니다. 따라서 이전 빌드에서 이미 다운로드되거나 설치된 의존성들에 대해서는 빠르게 처리할 수 있게 됩니다.

---

<aside>
💡 그리고 ECS service update가 될 때 마다 다른 컨테이너에 뜨긴하지만 이미지가 빌드되는 곳을 해당 컨테이너가 아니라 jenkins container에서 되는거니까 캐싱이 되는거고?

</aside>

네, 맞습니다. ECS에서 서비스가 업데이트될 때 새로운 태스크 (컨테이너)가 시작되지만, 이미지의 빌드 자체는 Jenkins에서 이루어집니다. 따라서 Jenkins에서의 이미지 빌드 과정에서 BuildKit의 캐싱 메커니즘이 활용되어 빌드 시간을 단축시킬 수 있습니다.

ECS의 컨테이너는 이미 빌드된 이미지를 바탕으로 시작되므로, 이미지 빌드 시점에서의 캐싱은 Jenkins에서만 중요합니다. ECS에서의 컨테이너 시작은 이미지 빌드와 무관하게, 해당 이미지를 기반으로 실행됩니다.

---

<aside>
💡 그럼 

RUN --mount=type=cache,target=/root/.npm npm ci

에서 root는 jenkins container를 의미하는거야?

</aside>

**`RUN --mount=type=cache,target=/root/.npm npm ci`** 명령어에서의 **`/root/.npm`**는 Docker 이미지 빌드 중인 임시 컨테이너의 파일 시스템 경로를 가리킵니다.

즉, **`--mount=type=cache,target=/root/.npm`** 부분은 Docker BuildKit의 캐싱 메커니즘을 활용하여 해당 임시 컨테이너의 **`/root/.npm`** 디렉토리에 캐시를 저장하라는 의미입니다.

Jenkins container와는 직접적인 연관이 없습니다. Jenkins는 단순히 이 Dockerfile을 사용하여 이미지를 빌드하는 역할을 합니다. **`/root/.npm`** 경로는 Docker 이미지 빌드 과정에서만 사용되는 임시 경로입니다

---

<aside>
💡 -mount=type=cache 는 jenkins container에 캐싱하겠다는 의미야? 캐싱된 데이터를 확인하려면 어디서확인할 수 있어?

</aside>

- **`-mount=type=cache`**는 Docker BuildKit의 캐싱 메커니즘을 활용하여 이미지 빌드 중 발생하는 데이터를 캐시하겠다는 의미입니다. 그러나 이 캐시는 Jenkins 컨테이너의 일부가 아닙니다. 대신 BuildKit에서 관리하는 별도의 캐시 저장소에 저장됩니다.

캐시된 데이터를 직접 확인하려면 일반적으로 Docker의 내부 메커니즘을 통해서는 어렵습니다. 하지만, Docker의 BuildKit 캐시 저장소를 확인하거나 관리하려면 명령어라인에서 도구와 명령어들을 사용할 수 있습니다. 그러나 이것은 고급 사용자를 위한 것이며 일반적으로 필요하지 않을 수 있습니다.

만약 캐시를 완전히 제거하고 싶다면, Docker의 이미지나 컨테이너를 제거하는 명령어들 (**`docker system prune`** 등)을 사용하여 관련 리소스를 정리할 수 있습니다. 그러나 이렇게 하면 모든 빌드 캐시가 삭제되므로, 다음 번 빌드에서 캐시의 장점을 활용할 수 없게 됩니다.

---

<aside>
💡 지금 이해가 안 된것은 jenkins 컨테이너에 캐싱되지 않는데 어떻게 이전 배포에서 캐싱된 것을 기억할 수 있는거야?

</aside>

좋은 질문입니다. 이해하기 위해 Docker BuildKit의 캐싱 메커니즘을 조금 더 깊이 이해해야 합니다.

Docker BuildKit의 캐싱 메커니즘은 일반적인 Docker 이미지 빌드와는 약간 다릅니다. BuildKit이 이미지를 빌드할 때 각 단계의 결과를 별도로 캐시에 저장합니다. 이 캐시는 Docker의 내부 저장소에 저장되며, Jenkins 컨테이너와는 별도로 관리됩니다. 따라서 Jenkins 컨테이너가 재시작되거나 변경되더라도 BuildKit의 캐시는 Docker 호스트에 유지됩니다.

즉, 이전 빌드에서 **`RUN --mount=type=cache,target=/root/.npm npm ci`** 명령을 수행했을 때 해당 단계의 결과물이 캐시에 저장되었다면, 다음 빌드에서 동일한 명령이 실행될 때 BuildKit은 이전에 저장한 캐시를 확인하고 재사용합니다. 이는 Jenkins 컨테이너의 상태와는 무관하게 동작합니다.

따라서, 질문의 답은 Docker 호스트의 내부 저장소에 저장된 BuildKit의 캐시 메커니즘 덕분에 이전 배포에서 캐싱된 것을 기억하고 재사용할 수 있습니다.

---

<aside>
💡 지금 상황이

jenkins는 EC2 instance에 container 형태로 띄워져 있어

너가 말한 Docker 호스트의 내부 저장소는 EC2 instance에 설치된 docker를 의미하는거야?

</aside>

네, 맞습니다.

Docker는 설치된 환경에서 컨테이너들을 관리하기 위한 데몬 프로세스(**`dockerd`**)를 실행합니다. 이 데몬 프로세스는 해당 시스템에 설치된 Docker의 주요 구성요소입니다. Jenkins가 EC2 인스턴스 내의 컨테이너 형태로 실행되고 있다면, 해당 EC2 인스턴스에 설치된 Docker는 그 데몬 프로세스와 관련된 모든 작업을 처리하게 됩니다.

이 Docker 데몬은 컨테이너의 실행, 이미지의 관리, 그리고 BuildKit과 같은 기능의 처리를 담당하며, BuildKit의 캐싱 메커니즘이나 다른 관련 데이터도 이 Docker 데몬이 관리하는 영역에 저장됩니다.

따라서, "Docker 호스트의 내부 저장소"라고 표현했을 때, 이는 Jenkins가 실행되고 있는 EC2 인스턴스에 설치된 Docker의 저장 영역을 의미합니다.

---