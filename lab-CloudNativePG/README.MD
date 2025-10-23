
# **Lab CloudNativePG dengan Local-Path, TLS, dan Pooler**

## **1. Persiapan Local-Path Storage**

Local-Path Storage dipakai sebagai Persistent Volume (PV) untuk PostgreSQL di lab.

### **Install Local-Path Provisioner**

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### **Cek StorageClass dan Pod**

```bash
kubectl get storageclass
kubectl get pods -n local-path-storage
```

* StorageClass default: `local-path`.
* PVC untuk cluster akan otomatis dibuat oleh CNPG operator menggunakan storageClass `local-path`.

---

## **2. Superuser Secret (`secret.yaml`)**

Secret ini berisi username dan password superuser PostgreSQL:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-example-superuser
type: Opaque
stringData:
  username: postgres
  password: SynapsisChall2025
```

* Apply:

```bash
kubectl apply -f secret.yaml
```

* Digunakan oleh cluster PostgreSQL untuk bootstrap database `appdb`.

---

## **3. Self-Signed Issuer (`issuer.yaml`)**

Issuer ini digunakan `cert-manager` untuk generate TLS certificate otomatis.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

* Apply:

```bash
kubectl apply -f issuer.yaml
```

---

## **4. Server TLS Certificate (`server-tls.yaml`)**

Certificate untuk PostgreSQL server, otomatis dibuat oleh `cert-manager` menggunakan issuer sebelumnya:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cluster-example-server-tls
spec:
  secretName: cluster-example-server-tls
  commonName: cluster-example.default.svc
  dnsNames:
    - cluster-example
    - cluster-example.default
    - cluster-example.default.svc
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
    group: cert-manager.io
  privateKey:
    algorithm: ECDSA
    size: 256
  usages:
    - server auth
    - client auth
```

* Apply:

```bash
kubectl apply -f server-tls.yaml
```

* Setelah berhasil, Secret `cluster-example-server-tls` akan terbentuk.

---

## **5. PostgreSQL Cluster (`cluster.yaml`)**

Cluster PostgreSQL dengan 2 instance, storage lokal, TLS dan superuser secret:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:13
  storage:
    size: 10Gi
    storageClass: local-path
  bootstrap:
    initdb:
      database: appdb
      owner: postgres
      secret:
        name: cluster-example-superuser
  postgresql:
    parameters:
      max_connections: "1000"
      pgaudit.log: "all, -misc"
      pgaudit.log_catalog: "off"
      pgaudit.log_parameter: "on"
      pgaudit.log_relation: "on"
  superuserSecret:
    name: cluster-example-superuser
  certificates:
    serverTLSSecret: cluster-example-server-tls
    serverCASecret: cluster-example-server-tls
    clientCASecret: cluster-example-server-tls
    replicationTLSSecret: cluster-example-server-tls
```

* Apply:

```bash
kubectl apply -f cluster.yaml
```

* CNPG operator otomatis membuat:

  * 2 pod PostgreSQL
  * 2 PVC 10Gi per pod
  * TLS Secret digunakan otomatis

---

## **6. NodePort Service (`nodeport.yaml`)**

Agar cluster dapat diakses dari luar node:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cluster-example-nodeport
spec:
  type: NodePort
  selector:
    cnpg.io/cluster: cluster-example
    role: primary
  ports:
    - port: 5432
      targetPort: 5432
      nodePort: 30432
```

* Apply:

```bash
kubectl apply -f nodeport.yaml
```

---

## **7. Pooler (`pooler-nodeport.yaml`)**

PgBouncer untuk connection pooling:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pooler-example-rw
spec:
  cluster:
    name: cluster-example
  instances: 1
  type: rw
  serviceTemplate:
    spec:
      type: NodePort
  pgbouncer:
    poolMode: session
    parameters:
      max_client_conn: "1000"
      default_pool_size: "10"
```

* Apply:

```bash
kubectl apply -f pooler-nodeport.yaml
```

---

## **8. Pod Client untuk Testing (`psql-client-tls.yaml`)**

Pod ini digunakan untuk testing koneksi TLS:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: psql-client-tls
spec:
  restartPolicy: Never
  containers:
    - name: psql
      image: postgres:13
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: tls-secret
          mountPath: /tls
          readOnly: true
  volumes:
    - name: tls-secret
      secret:
        secretName: cluster-example-server-tls
```

* Apply:

```bash
kubectl apply -f psql-client-tls.yaml
```

* Masuk ke Pod:

```bash
kubectl exec -it psql-client-tls -- bash
```

* Tes koneksi TLS:

```bash
psql "host=cluster-example-rw.default.svc.cluster.local dbname=appdb user=postgres sslmode=verify-ca sslrootcert=/tls/tls.crt"
```

* Password: `SynapsisChall2025` (dari secret)

---

## **9. Testing & Troubleshooting**

* **Cek pod & service**

```bash
kubectl get pods
kubectl get svc
kubectl get secret
```

* **Cek certificate**

```bash
kubectl get certificate cluster-example-server-tls
kubectl describe certificate cluster-example-server-tls
```

* **Jika koneksi gagal**

  * Pastikan nama host sesuai `cluster-example-rw.default.svc.cluster.local`
  * Pastikan file `/tls/tls.crt` ada di Pod client
  * Pastikan password superuser benar sesuai secret

---

