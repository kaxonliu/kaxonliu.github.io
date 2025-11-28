# webhook

Admission Webhooks 是一种用于扩展和自定义 Kubernetes API 服务器行为的机制。简单来说，它们是一种**回调机制**，允许你在 API 服务器处理请求的**关键节点**上，拦截请求并进行自定义的验证或修改。



## 在 API 请求流程中的位置

要理解 Webhooks，首先要知道一个 Kubernetes API 请求的生命周期：

~~~bash
1.认证 -> 2.鉴权 -> 3.准入控制 -> 4.持久化
~~~

Admission Webhooks 就工作在 **第 3 步：准入控制** 阶段。这个阶段又分为两个子阶段：

- **Mutating Admission Webhook**：**先执行**。它可以**修改**到达 API 服务器的对象。例如，为 Pod 自动注入一个 Sidecar 容器、添加特定的标签或环境变量。
- **Validating Admission Webhook**：**后执行**。它只能**验证**对象，不能修改。例如，检查部署的镜像是否来自可信的仓库，或确保资源请求和限制被正确设置。

**重要顺序**：Mutating Webhooks 执行完毕后，API 对象才会被交给 Validating Webhooks 进行验证。这确保了验证逻辑是针对最终将要存储的对象进行的。



## 工作原理

1. **用户发起请求**： 比如，使用 `kubectl apply -f pod.yaml` 创建一个 Pod。
2. **API 服务器拦截**： API 服务器接收到请求，在完成认证和鉴权后，进入准入控制阶段。
3. **查询 Webhook 配置**： API 服务器检查集群中是否存在与当前请求相关的 `MutatingWebhookConfiguration` 或 `ValidatingWebhookConfiguration` 资源。
4. **发送 AdmissionReview 请求**： 如果配置匹配（例如，针对 `v1` 版本的 `pods` 资源，执行 `CREATE` 操作），API 服务器会向 Webhook 配置中指定的外部 HTTP(S) 服务（你的 Webhook 服务器）发送一个 `AdmissionReview` 请求。这个请求中包含了待处理对象的详细信息。
5. **Webhook 服务器处理**：
   - **Mutating Webhook**： 可以修改对象，并生成一个 JSON Patch 返回给 API 服务器。
   - **Validating Webhook**： 检查对象，然后决定是“允许”还是“拒绝”该请求。
6. **API 服务器响应**：
   - 如果 Webhook 返回 `"allowed": true`，请求继续向下执行。
   - 如果 Webhook 返回 `"allowed": false`，整个请求将被立即拒绝，并返回一个错误给用户。
   - 对于 Mutating Webhook，API 服务器会应用返回的 JSON Patch 来更新对象。



## Webhook 配置资源

想要使用 webhook，需要在 k8s 集群中定义 webhook 资源（用来主动触发钩子功能的请求），资源分为两类：

- 修改数据的资源 MutatingWebhookConfiguration
- 校验数据的资源 ValidatingWebhookConfiguration



#### MutatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-mutating-webhook
webhooks:
- name: my-webhook.example.com
  clientConfig:
    # 你的 Webhook 服务器的地址
    service:
      namespace: "webhook-namespace"
      name: "webhook-service"
      path: "/mutate" # 服务器处理 mutate 请求的路径
      port: 443
    caBundle: <CA_BUNDLE> # CA 证书，用于验证服务器证书
  rules: # 定义哪些请求需要被拦截
  - operations: ["CREATE", "UPDATE"] # 操作类型
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"] # 资源类型
  admissionReviewVersions: ["v1"] # 支持的 AdmissionReview 版本
  sideEffects: None # 声明该 Webhook 是否有副作用
  timeoutSeconds: 30 # 超时时间
```

#### ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-validating-webhook
webhooks:
- name: my-validating-webhook.example.com
  clientConfig:
    service:
      namespace: "webhook-namespace"
      name: "webhook-service"
      path: "/validate"
      port: 443
    caBundle: <CA_BUNDLE>
  rules:
  - operations: ["CREATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 10
```



**关键字段解释**：

- `clientConfig`： 告诉 API 服务器如何访问你的 Webhook 服务。可以是 Kubernetes `Service`，也可以直接是 `url`。
- `rules`： 一组规则，用于匹配哪些请求应该被发送到 Webhook。非常灵活，可以按 API Group、Version、Resource 和 Operation 进行过滤。
- `caBundle`： **非常重要**。这是一个 PEM 编码的 CA 证书包，用于验证 Webhook 服务器的 TLS 证书。这确保了 API 服务器与一个可信的端点通信。
- `sideEffects`： 声明 Webhook 除了处理 AdmissionReview 之外，是否会对集群外部系统产生副作用（例如发送邮件）。如果设置为 `None` 或 `NoneOnDryRun`，在某些情况下（如 `kubectl apply --dry-run=server`）可以安全地跳过该 Webhook，提高性能。





## mutating webhook 示例

需求：为新创建的 pod 打上标签：environment: production

具体实现可以兵分两路，一路准备 webhook 资源；一路准备 web 应用实现业务逻辑。在开始操作前需要先定义好两者的交互格式。

#### 请求格式

mutating webhook 收到创建 pod 的数据后，主动向 web 应用发送一个 POST 请求，请求体中的数据是固定的。需要定义 web 应用的 URL

#### 响应格式

响应数据格式为 json，修改的内容放在 `patch` 字段的值上，Base64 编码。

~~~json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<请求中的 uid>", // 必须与请求中的 UID 匹配
    "allowed": true, // 或 false
    "status": {
      "code": 403, // 如果 allowed 为 false，可以设置 HTTP 状态码
      "message": "解释为什么被拒绝" // 给用户看的错误信息
    },
    // --- 仅对 Mutating Webhook 需要 ---
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0=" // Base64 编码的 JSON Patch
  }
}
~~~



#### 具体安排

使用 python flask web框架实现一个 web 服务，该 web 服务提供 URI （`/mutate`）用于提供具体业务需求。这个 web 服务的协议需要是 https 的，因此需要为这个服务自建 ssl 证书。自建 ssl 证书前需要准备好服务的域名。

方便起见，可以在 k8s 集群中把 web 服务放在 pod 中运行，使用 deployment 控制器资源管理，配合 service 资源，最终可以对外暴露一个 固定域名的服务。

自建 ssl 证书时就使用这个 svc 域名，同时创建 `MutatingWebhookConfiguration` 也需要使用这个 service na me。



#### 第一步：开发 web 应用

使用 flask 框架开发，定义 `mutate` 接口，内部实现业务需求，配置使用 ssl 证书。

~~~python
# webhook.py
import base64
import json
import ssl

from flask import Flask, request, jsonify

app = Flask(__name__)


def create_patch(metadata):
    """
    创建 JSON Patch 以添加 'mutate' 注释。
    如果 metadata.annotations 不存在，则首先创建该路径。
    """

    dic = {}
    if 'labels' in metadata:
        dic = metadata['labels']

    patch = [
        {'op': 'add', 'path': '/metadata/labels', 'value': dic},
        {'op': 'add', 'path': '/metadata/labels/environment', 'value': 'production'}
    ]
    patch_json = json.dumps(patch)
    patch_base64 = base64.b64encode(patch_json.encode('utf-8')).decode('utf-8')
    return patch_base64


# https://webhook-service.default.svc:443/mutate
@app.route('/mutate', methods=['POST'])
def mutate():
    """
    处理 Mutating Webhook 的请求，对 Pod 对象应用 JSON Patch。
    """
    # 从请求中提取 AdmissionReview 对象
    admission_review = request.get_json()

    # 验证 AdmissionReview 格式是否正确
    # admission_review['request']['object']
    if 'request' not in admission_review or 'object' not in admission_review['request']:
        return jsonify({
            'kind': 'AdmissionReview',
            'apiVersion': 'admission.k8s.io/v1',
            'response': {
                'allowed': False,  # 如果格式无效，则禁止当前提交过来的资源请求
                'status': {'message': 'Invalid AdmissionReview format'}
            }
        })

    req = admission_review['request']  # 提取请求对象
    print('--->', req)
    # 生成 JSON Patch
    metadata = req['object']['metadata']
    patch_json = create_patch(metadata)

    # 准备 AdmissionResponse 响应
    admission_response = {
        'kind': 'AdmissionReview',
        'apiVersion': 'admission.k8s.io/v1',
        'response': {
            'uid': req['uid'],
            'allowed': True,
            'patchType': 'JSONPatch',
            'patch': patch_json  # 直接包含 Patch 数据作为 JSON 字符串
        }
    }
    print(admission_response)
    return jsonify(admission_response)


if __name__ == '__main__':
    # 加载 SSL 证书和私钥
    context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
    context.load_cert_chain('/certs/tls.crt', '/certs/tls.key')

    # Run the Flask application with SSL
    app.run(host='0.0.0.0', port=443, ssl_context=context)
~~~



#### 第二步：制作镜像

准备 dockerfile 文件

~~~dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY webhook.py .

RUN pip install Flask -i https://mirrors.ustc.edu.cn/pypi/web/simple

CMD ["python", "webhook.py"]
~~~

构建镜像

~~~bash
docker build -f dockerfile -t mute-webhook:v1.0 .
~~~

打标签上传自建镜像仓库

~~~bash
# 打标签
docker tag mute-webhook:v1.0 registry...aliyuncs.com/l.7/mute-webhook:v1.0
# 上传
docker push registry...aliyuncs.com/l.7/mute-webhook:v1.0
~~~



#### 第三步：生成 ssl 证书

**生成 CA 私钥**，得到 `ca.key` 文件。

~~~bash
openssl genrsa -out ca.key 2048
~~~

**生成自签名 CA 证书**，有效期为 100 年。这里需要指定 web 服务的域名，后面部署 svc 时需要和这个名称保持一致。得到 `ca.crt` 文件。

~~~bash
openssl req -x509 -new -nodes -key ca.key -subj "/CN=webhook-service.default.svc" -days 36500 -out ca.crt
~~~

创建证书请求的配置文件

~~~bash
cat > webhook-openssl.cnf << 'EOF'
[req]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C = CN
ST = Shanghai
L = Shanghai
O = kaxonliu
OU = kaxonliu
CN = webhook-service.default.svc

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = webhook-service
DNS.2 = webhook-service.default
DNS.3 = webhook-service.default.svc
DNS.4 = webhook-service.default.svc.cluster.local


[req_distinguished_name]
CN = webhook-service.default.svc

[v3_req]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[v3_ext]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names

EOF
~~~

生成 Webhook 服务的私钥，得到 `webhook.key` 文件。

~~~bash
openssl genrsa -out webhook.key 2048
~~~

使用 OpenSSL 配置文件生成 CSR，得到 `webhook.csr` 文件。

~~~bash
openssl req -new -key webhook.key -out webhook.csr -config webhook-openssl.cnf
~~~

使用 ca 证书签署 webhook 服务证书，得到 `webhook.crt` 文件。这之后上一步生成的 `webhook.csr` 文件就不再需要了。

~~~bash
openssl x509 -req -in webhook.csr -signkey webhook.key -out webhook.crt -days 365

openssl x509 -req -in webhook.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out webhook.crt -days 36500 -extensions v3_ext -extfile webhook-openssl.cnf
~~~

最终得到 webhook.crt、webhook.key，基于这俩文件创建 secret 资源。

#### 第四步：创建 secret 资源

将生成的证书和私钥存储在 Kubernetes Secret 中。

~~~bash
kubectl create secret tls webhook-certs \
--cert=webhook.crt \
--key=webhook.key \
--namespace=default \
--dry-run=client -o yaml | kubectl apply -f -
~~~

查看 secret 资源

~~~bash
[root@k8s-master-01 work]# kubectl get secrets 
NAME            TYPE                DATA   AGE
webhook-certs   kubernetes.io/tls   2      18s
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# kubectl describe secrets webhook-certs
Name:         webhook-certs
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.key:  1675 bytes
tls.crt:  1265 bytes
~~~



#### 第五步：创建 deployment 来部署 webhook 服务

创建如下 yaml 文件，包含 deployment 和 service 的配置清单。

~~~yaml
# webhook-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      containers:
      - name: webhook
        image: registry...aliyuncs.com/l.7/mute-webhook:v1.0	# 镜像
        volumeMounts:
        - name: webhook-certs
          mountPath: /certs
          readOnly: true
      volumes:
      - name: webhook-certs
        secret:
          secretName: webhook-certs
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-service		# 这个名字要和自建证书时使用的域名保持一致
  namespace: default
spec:
  ports:
  - port: 443
    targetPort: 443
  selector:
    app: webhook
~~~



#### 第六步：创建 webhook 资源

基于脚本来生成 MutatingWebhookConfiguration 资源的 yaml 文件。（这样可以避免 证书文件内容拷贝出错

~~~bash
#!/bin/bash

base64 -w 0 ca.crt > ca.crt.base64

# 定义文件路径
ca_base64_file="ca.crt.base64"
yaml_file="m-w-c.yaml"

# 读取 ca.crt.base64 的内容
ca_base64_content=$(cat "$ca_base64_file" | tr -d '\n')

# 生成替换后的 YAML 文件内容
# 将 base64 内容插入到 YAML 文件中
cat <<EOF > "$yaml_file"
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: example-mutating-webhook
webhooks:
- name: example.webhook.com
  clientConfig:
    service:
      name: webhook-service
      namespace: default
      path: "/mutate"
      port: 443  # 添加端口号，通常是 443
    # 替换为  base64 编码的 CA 证书
    caBundle: "$ca_base64_content"
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
EOF

echo "YAML 文件已更新。"
~~~

执行脚本，创建 yaml 文件，得到文件 `m-w-c.yaml` 

~~~bash
bash create-mute-webhook.sh
~~~



#### 第七步：创建资源

创建 MutatingWebhookConfiguration 资源

~~~bash
[root@k8s-master-01 work]# kubectl apply -f m-w-c.yaml 
mutatingwebhookconfiguration.admissionregistration.k8s.io/example-mutating-webhook created
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# kubectl get mutatingwebhookconfigurations.admissionregistration.k8s.io 
NAME                       WEBHOOKS   AGE
example-mutating-webhook   1          11s
~~~



#### 第八步：测试

创建 pod 测试，发现新建的 pod 自动添加标签 `environment=production`

~~~bash
[root@k8s-master-01 work]# kubectl run test-pod --image=nginx --dry-run=client -o yaml |kubectl apply -f -
pod/test-pod created
[root@k8s-master-01 work]# 
[root@k8s-master-01 work]# kubectl get pod test-pod --show-labels 
NAME       READY   STATUS    RESTARTS   AGE   LABELS
test-pod   1/1     Running   0          27s   environment=production,run=test-pod
~~~



## validate webhook 示例

整体思路和 mutating webhook 的使用一致。使用 `ValidatingWebhookConfiguration` 资源。创建的 web 程序如下。

~~~python
# 校验新建 pod 必须要有 'environment' 标签
import logging
import ssl

from flask import Flask, request, jsonify


app = Flask(__name__)
logging.basicConfig(level=logging.INFO)


@app.route('/validate', methods=['POST'])
def validate():
    admission_review = request.get_json()
		# 校验失败的返回格式和内容
    if 'request' not in admission_review or 'object' not in admission_review['request']:
        return jsonify({
            'kind': 'AdmissionReview',
            'apiVersion': 'admission.k8s.io/v1',
            'response': {
                'allowed': False,
                'status': {'message': 'Invalid AdmissionReview format'}
            }
        })

    req = admission_review['request']

    # 只处理 Pod
    if req['kind']['kind'] == 'Pod':
        pod = req['object']
        labels = pod.get('metadata', {}).get('labels', {})

        # 检查是否有 'environment' 标签
        if 'environment' not in labels:
            return jsonify({
                'kind': 'AdmissionReview',
                'apiVersion': 'admission.k8s.io/v1',
                'response': {
                    'uid': req['uid'],
                    'allowed': False,
                    'status': {
                        'metadata': {},
                        'code': 400,
                        'message': 'Pod must have an "environment" label'
                    }
                }
            })

        return jsonify({
            'kind': 'AdmissionReview',
            'apiVersion': 'admission.k8s.io/v1',
            'response': {
                'uid': req['uid'],
                'allowed': True,
                'status': {
                    'metadata': {},
                    'code': 200
                }
            }
        })

    return jsonify({
        'kind': 'AdmissionReview',
        'apiVersion': 'admission.k8s.io/v1',
        'response': {
            'allowed': True,
            'status': {
                'metadata': {},
                'code': 200
            }
        }
    })

if __name__ == '__main__':
    context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
    context.load_cert_chain('/certs/tls.crt', '/certs/tls.key')
    app.run(host='0.0.0.0', port=443, ssl_context=context)
~~~

