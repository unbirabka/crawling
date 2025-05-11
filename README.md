# Hello MrScraper
selamat datang, repo ini saya buat untuk memenuhi request "MrScraper"
untuk proses deployment nya saya menggunakan ci/cd `github-action`, karena disini saya menggunakan minikube, saya hanya menggunakan `github-action` untuk keperluan generate manifest k8s dan build image docker, ada 2 methode, `manual` dan `otomatis` ketika github mendetech terjadi perubahan pada folder `service-a` di branch `main`

untuk manual bisa mengikuti step dibawah ini

- https://github.com/unbirabka/crawling > action > service-a > `run workflow` pilih branch `main`

untuk otomatis bisa mengikuti step dibawah ini

- changes file atau menambahkan atau apapun yg ada didalam folder `service-a/` dan kemudian push ke branch main, atau buat pr ke branch main, karena metode otomatis ini hanya bekerja jika folder tersebut berubah di branch `main`

## Building and pushing the Docker image.

untuk proses build dan push , saya menggunakan kurang lebih 2 step dibawah pada stage `build_docker_image` yang pertama untuk create auth ke dockerhub, lalu build menggunakan file `Dockerfile` yang berada di path `service-a/Dockerfile` dan kemudian push ke repo `sekolahlinux/crawler` dengan tag `service-a-${{env.SHORT_SHA}}`, yang mana `${{env.SHORT_SHA}}` diambil dari 8 char sha commit

```
      - name: Get Short SHA
        id: short-sha
        run: |
          echo "SHORT_SHA=`echo ${GITHUB_SHA} | cut -c1-8`" >> $GITHUB_ENV
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: sekolahlinux
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and Push
        uses: docker/build-push-action@v4
        with:
          file: service-a/Dockerfile
          push: true
          tags: sekolahlinux/crawler:service-a-${{env.SHORT_SHA}}
```

## Setting up the Kubernetes cluster.

untuk proses setup kubernetes saya menggunakan minikube di laptop macbook m1 saya dengan menjalankan perintah dibawah ini
```
minikube start --memory=10384 --cpus=4 --kubernetes-version=v1.32.4
```

lalu saya juga setup beberapa dependency terkait monitoring menggunakan helm, untuk values yaml helm nya sendiri saya letaknya di folder `helm` pada root dir git repo ini, beberapa dependency yang saya install antara lain

- kibana
```
helm upgrade -i -f kibana**-8.5.1.yaml kibana elastic/kibana --version 8.5.1
```
- elasticsearch
```
helm upgrade -i -f elasticsearch-8.5.1.yaml elasticsearch elastic/elasticsearch --version 8.5.1
```
- grafana
```
helm upgrade -i -f values-8.13.1.yaml grafana grafana/grafana --version 8.13.1
```
- prometheus
```
helm upgrade -i -f values-27.11.0.yaml prometheus prometheus-community/prometheus --version 27.11.0
```
- keda
```
helm upgrade -i -f values-2.17.0.yaml keda kedacore/keda --version 2.17.0 --namespace keda --create-namespace
```
- fluentbit
```
helm upgrade -i -f values-0.49.0.yaml fluent-bit fluent/fluent-bit --version 0.49.0 
```
- ingress-nginx
```
- helm upgrade -i -f values-4.12.2.yaml ingress-nginx ingress-nginx/ingress-nginx --version 4.12.2 --namespace ingress-nginx
```
- metric-server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```


## Applying the YAML

saya menggunakan helm untuk k8s package managementnya, untuk helm values nya sendiri saya put disini

- https://github.com/unbirabka/crawling/blob/main/service-a/values.yaml.bak

proses generate nya ada pada step dibawah ini pada stage `deploy_k8s`
```
      - name: Helm Values Preparation
        run: |
          envsubst < service-a/values.yaml.bak > service-a/values.yaml
      - name: Helm Apply
        run: |
          aws --version
          helm version
          helm repo add sekolahlinux 'https://raw.githubusercontent.com/unbirabka/helm-sekolahlinux/main/charts/'
          helm repo update
          helm template service-a sekolahlinux/sekolahlinux --version=0.1.0 -f service-a/values.yaml > service-a-${{env.SHORT_SHA}}.yaml
      - name: 'Upload Artifact'
        uses: actions/upload-artifact@v4
        with:
          name: artifact manifest k8s service-a
          path: service-a-${{env.SHORT_SHA}}.yaml
          retention-days: 3
```
hasil dari menjalankan github action diatas, akan generate file `service-a-****.yaml` nanti nya file tersebut akan di upload ke artifact dan kemudian bisa di download dan manifest nya bisa di apply manual dengan `kubectl apply` pada minikube cluster, untuk proses generate manifest saya menggunakan helm chart yang saya buat sendiri, dan bisa dilihat pada repo dibawah ini

- https://github.com/unbirabka/helm-sekolahlinux

contoh file manifest yang sudah di generate bisa di lihat pada file ini

- https://github.com/unbirabka/crawling/blob/main/manifest-k8s/service-a.yaml

di dalam manifest diatas sudah terdapat beberapa module k8s diantaranya

- keda
- deployment
- service
- ingress

in case jika menggunakan aws eks, saya juga put pada github-action workflows yaml, namun saya comment karena saya disini hanya perlu untuk generate manifest k8s nya dan apply manual ke minikube, selain itu saya juga sedikit menambahkan unit test linter sedernaha, untuk detail config workflownya ada disini

- https://github.com/unbirabka/crawling/blob/main/.github/workflows/service-a.yml

beberapa task yang saya kerjakan diatas juga pernah saya kerjakan baik di current company ataupun di sebelumnya, dan bisa dibaca di

- https://sekolahlinux.com/ 
