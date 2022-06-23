# KONG_K8S-Ingress-PostgreSQL

 Triển khai API Gateway lên Kubernetes với Kong API Gateway, Konga và PostgreSQL

Chào mọi người, lại là mình đây. Tiếp tục chuỗi bài viết tìm hiểu về Kubernetes và Microservices Archiecture, bài viết hôm nay chúng ta cùng triển khai thực tế một API gateway lên Kubernetes thông qua ví dụ cụ thể nhé

Bài viết trước chúng ta đã tìm hiểu về vai trò và tầm quan trọng của API gateway trong một hệ thống Microservices, mọi người có thể tìm đọc ở đây https://viblo.asia/p/tap-tanh-microservices-voi-kong-api-gateway-va-docker-compose-1VgZvJjrZAw

Bài viết hôm nay chúng ta sẽ thực hành các bước để triển khai API Gateway lên môi trường Kubernetes với Kong, Konga, PostgreSQL. Và để làm được điều này, hãy cùng mình điểm qua một vài khái niệm sau

    StatefulSet
    Ingress
    Persistent Volumes
    Persistent Volume Claims

StatefulSet

https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

StatefulSet là một workload của K8s dùng để chạy một Ứng dụng Stateful - Ứng dụng cần lưu lại toàn bộ dữ liệu và không bị mất đi sau quá trình re-create

StatefulSet được sử dụng để deploy các ứng dụng dạng này, và trong bài viết này mình sẽ dùng StatefulSet để deploy PostgreSQL. StatefulSet quản lý deployment và scale các pods, nó cung cấp sự đảm bảo về thứ tự và tính duy nhất của các Pods này.

Cách quản lý các Pods của Deployment và StatefulSet về cơ bản là giống nhau. Nhưng không như Deployment, StatefulSet duy trì tính độc nhất và cố định cho các pods của nó. Các pods này không thể hoán đổi cho nhau.
Ingress

Ingress là một api trong K8s, quản lý truy cập từ bên ngoài vào các services trong Cluster thông qua http.

Ingress tiếp nhận request HTTP, HTTPS từ bên ngoài và điều hướng đến các services bên trong cluster. Router được điều khiển và điều hướng thông qua rules được định nghĩa khi triển khai Ingress

image.png

Với ingress, chúng ta có thể load balancing, SSL termination và định danh các services qua name-based.
Persistent Volumes

    A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual Pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.

Persistent Volume (PV) là không gian lưu trữ trong cluster, được cấp phát bởi Admin hoặc được cấp phát động. Nó là một dạng resource trong cluster như là node, so sánh thì PV đại diện tài nguyên lưu trữ, còn node thì là tài nguyên ram, cpu ...
Persistent Volume Claims

    A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany, see AccessModes).

Persistent Volume Claim (PVC) là một request để yêu cầu sử dụng không gian lưu trữ từ người dùng đối với một PV nhất định. Claim có thể yêu cầu kích thước không gian lưu trữ cũng như có thể yêu cầu về quyền truy cập không gian đó như ReadWriteOnce, ReadOnlyMany hoặc ReadWriteMany ...

Container trong pods sẽ cần không gian lưu trữ cho các dạng dữ liệu cố định, như là static files, files upload từ người dùng, hoặc quan trọng nhất chính là Databases, đều được lưu trữ trong PV thông qua khai báo PVC.
Các bước triển khai

Giới thiệu vậy đủ rồi, cùng tới phần triển khai xem các bước sẽ như thế nào nhé.

Để mọi người dễ hình dung, mình sẽ deploy các thành phần cơ bản của một Microservices lên Kubernetes.

Để làm được điều này thì trước tiên cần chuẩn bị môi trường Kubernetes trước đó, mọi người có thể tìm đọc qua các bài trước của series này nhé.

    Khởi tạo Persistent Volume và Persistent Volume Claim
    Deploy StatefulSets nhằm mục đích chạy các databases
    Deploy 2 microservices là json-api-service, users-service
    Deploy Ingress cho các microservices nhằm định tuyến bên trong cụm
    Deploy API Gateway - Kong và Konga để quản lý API Gateway qua UI
    Proxy các port cho Konga và tiến hành routing các services

Lưu ý: Bài viết sẽ khá dài do một vài cấu hình mình viết trực tiếp trong bài viết, cụ thể về các cấu hình mình không đề cập trong bài viết mọi người có thể tham khảo ở đây : https://github.com/at-uytran/kubernetes
Khởi tạo Persistent Volume và Persistent Volume Claim

Chúng ta sẽ có 2 namespace cần lưu ý là namespace default và namespace kong, kong sẽ là nơi deploy cũng như phân biệt kong với các resource bình thường khác

Trong bài viết này mình sẽ demo cơ bản cho Kong namespace, namespace default để deploy các microservice mình sẽ không đề cập trong bài viết, các bạn có thể tham khảo qua source ở github nhé

    Deploy kong namespace thông qua câu lệnh kubectl apply -f kong_namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: kong

    Deploy PV và PVC: kubectl apply -f pg_pv_claim.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg-pv-volume
  namespace: kong
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: kong
  name: pg-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

Hãy kiểm tra thông qua giao diện xem các resouces này đã được khởi tạo hay chưa trước khi làm các bước tiếp theo nhé.

Deploy StatefulSets nhằm mục đích chạy các databases

Sau khi khởi tạo xong PV và PVC thì chúng ta cần có một instance chạy databases cho ứng dụng, khái niệm vì sao dùng statefulset thì mọi người có thể đọc lại khái niệm của nó nhé, chúng ta không thể deploy database như việc deploy một pod bình thường vì chúng ta không thể để database stop và start như cơm bữa phải không nào =))

Ở đây vì chỉ là demo nên thì mình sẽ để password trực tiếp trong đặc tả để đỡ rối nhé.

Ở file này mọi người hãy quan tâm đến block volumes, ở đây chúng ta đồng thời khai báo volumes cho statefulset này là pg-persistent-storage, sử dụng pvc là pg-pv-claim, và khai báo nó ở volumeMounts cho containers.

Bằng cách này thì dữ liệu trong container sẽ được mount vào PV và tồn tại cả khi tắt container và bật lại, các pods chạy cần lưu các dữ liệu cố định khác thì mọi người có thể deploy và dùng pvc riêng nhé.

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
  namespace: kong
spec:
  serviceName: postgresql-db-service
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
      - name: postgresql-db
        image: postgres:9.5
        volumeMounts:
        - name: pg-persistent-storage
          mountPath: /var/lib/postgresql/data
        env:
          - name: POSTGRES_PASSWORD
            value: kong
          - name: PGDATA
            value: /var/lib/postgresql/data/kong
      volumes:
      - name: pg-persistent-storage
        persistentVolumeClaim:
          claimName: pg-pv-claim

Tiếp theo chúng ta cần deploy service, để các service khác cũng như Kong API Gateway có thể gọi đến postgresql nhé

apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: kong
spec:
  ports:
  - port: 5432
  selector:
    app: postgresql-db
  clusterIP: None

Deploy 2 microservices là json-api-service, users-service

Deploy 2 microservice này sẽ chỉ đơn giản là một file đặc tả deployment, một file cho services. Việc deploy users-service sẽ tương tự cách mình deploy ở đây Deploy ứng dụng đầu tiên của bạn lên Kubernetes, tuy nhiên service bây giờ sẽ không cần dùng đến nodeport mà chỉ dùng service bình thường thôi

Sample cho json-api-service:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: json-api-deployment
  labels:
    app: json-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: json-api
  template:
    metadata:
      labels:
        app: json-api
    spec:
      imagePullSecrets:
        - name: docker-registry
      containers:
      - name: json-api
        image: uytran12/json-api:dev
        imagePullPolicy: Always
        ports:
        - containerPort: 4000
---
apiVersion: v1
kind: Service
metadata:
  name: json-api-service
  labels:
    app: json-api
spec:
  ports:
  - port: 4000
    name: http
    targetPort: 4000
  selector:
    app: json-api

Deploy Ingress cho các microservices nhằm định tuyến bên trong cụm

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: users-service
  namespace: default
 spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: users-service
            port:
              number: 3000

Hãy làm tương tự cho json-api-service nhé
Deploy API Gateway - Kong và Konga để quản lý API Gateway qua UI

Về Kong, chúng ta có 1 bài hướng dẫn chi tiết để deploy Kong lên K8S, tuy nhiên bài viết đó chỉ sử dụng db-less tức là không sử dụng database.

Ở bài viết này chúng ta sẽ sử dụng database nên cần chỉnh sửa các step cũng như chuẩn bị nhiều resouces hơn ví dụ như database prepare, database migration, các đặc tả deployment cũng cần chỉnh sửa để phù hợp với yêu cầu là có thể expose được giao diện Konga UI ra để thao tác với Kong dễ dàng hơn

    File yaml để deploy Kong resources mà không dùng database được Kong hướng dẫn ở đây https://bit.ly/k4k8s
    Tuy nhiên bài viết này mình sẽ không dùng cách này, hãy sửa một số thông tin so với file này

Ở các document của Kong và Konga, chúng ta có các hướng dẫn để chuẩn bị database khi chạy đơn hoặc chạy bằng docker-compose, tuy nhiên với Kubernetes thì mình không tìm thấy hướng dẫn nào.

Vậy ở đây các bước mà cần chuẩn bị trước khi chạy Kong và Konga là chuẩn bị database, migration database, nên mình sẽ chạy 2 service này với mục đích là khởi tạo database cho Kong và Konga

Cơ bản là sẽ giống các hướng dẫn khi deploy bằng docker-compose tuy nhiên cần viết để có thể deploy lên K8s nhé.

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kong-migration
  name: kong-migration
  namespace: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kong-migration
  template:
    metadata:
      annotations:
        kuma.io/gateway: enabled
        prometheus.io/port: "8100"
        prometheus.io/scrape: "true"
        traffic.sidecar.istio.io/includeInboundPorts: ""
      labels:
        app: kong-migration
    spec:
      containers:
      - env:
        - name: KONG_DATABASE
          value: "postgres"
        - name: KONG_PG_DATABASE
          value: "kong"
        - name: KONG_PG_HOST
          value: "postgres-db"
        - name: KONG_PG_USER
          value: "postgres"
        - name: KONG_PG_PASSWORD
          value: "kong"
        image: kong:2.5
        command: ["/bin/sh", "-c", "kong migrations bootstrap && kong migrations up && kong migrations finish"]
        name: kong-migration

    Deploy kong_all_in_one_custom.yaml - file này mình đã custom lại cho phù hợp với yêu cầu chạy Kong cùng database:

kubectl apply -f kong/kong_all_in_one_custom.yaml

Sau khi database cũng như Kong API đã lên thì hãy deploy Konga nhé

kubectl apply -f konga/konga_deployment.yaml kubectl apply -f konga/konga_service.yaml
Proxy các port cho Konga và tiến hành routing các services

Hãy kiểm tra các pods đang chạy ở all namespace xem sao, khi mọi pods đã chạy ok tức là app đã lên cơ bản rồi nhé mọi người, đến đây chúng ta có thể expose các port của Kong và Konga để demo rồi nhé. kubectl get pods --all-namespace

Vì là demo local nên mình không thể có được địa chỉ ip và external Ip address, nên chúng ta sẽ sử dụng cách forward port để demo nhé 😄

    Forward port cho Kong để truy cập api gateway qua port 8080

kubectl port-forward -n kong service/kong-proxy 8080:80

    Forward port cho Konga để truy cập konga UI qua port 1337

kubectl port-forward -n kong service/konga-svc 1337:1337

Các cách thao tác với Konga thì mọi người có thể tìm hiểu thông qua bài viết trước đây của mình https://viblo.asia/p/tap-tanh-microservices-voi-kong-api-gateway-va-docker-compose-1VgZvJjrZAw

Ở bước thao tác để tạo các services, khi sử dụng ingress, thì các service chúng ta có được định danh dns cơ bản như này Nếu chúng ta deploy foo-service như ví dụ ở document của Kong thì dns sẽ như này foo-service.default.3000.svc, các services có thể gọi qua nhau thông qua các dns này, tất nhiên là Kong API Gateway cũng vậy.

Format của ingressed name base này sẽ như sau: <tên service>.< namespace >.< port>.svc

Như vậy là mình vừa demo xong phần triển khai api gateway lên Kubernetes, bài viết hơi dài do các đoạn cấu hình resources.
