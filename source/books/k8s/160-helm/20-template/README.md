# 模板语法

~~~bash
一、内置对象

.Release:
	.Release.Name
	.Release.Namespace

	
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: {{ .Release.Name }}-deployment # xxx-deployment
	  namespace: {{ .Release.Namespace }}  # kube-system
	  labels:
		x: {{ .Release.IsUpgrade }}  # false

		y: {{ .Release.IsUpgrade }}  # false
		z: {{ .Release.IsInstall }}  # true
		a: {{ .Release.Revision }}   # 1
		b: {{ .Release.Service }}    # helm

	
	# 命令：
	helm install xxx mychart -n kube-system


.Values: 封装的是所有传入值的合集（
	--set传入
	自定义my_values.yaml
	chart包内自带的valus.yaml
	
	
	
.Chart
	{{ .Chart.Name }}
	{{ .Chart.Version }}
	{{ .Chart.AppVersion }}
	
	
	
	
.Files  获取文件的内容
	
	 containers:
	- name: {{ .Chart.Name }}
	  env:
		user: {{ .Files.Get "username.txt" | trim }}
		pass: {{ .Files.Get "password.txt" | trim }}


	
	
.Capabilities：提供了获取有关 Kubernetes 集群支持功能的信息的对象
	
	
	apiVersion: v1
	kind: Service
	metadata:
	  name: xxxx
	  labels:
		x: {{ .Capabilities.KubeVersion}}          # 结果：1.30.0
		y: {{ .Capabilities.KubeVersion.Major }}   # 结果：1
		z: {{ .Capabilities.KubeVersion.Minor }}   # 结果：30
		m: {{ .Capabilities.KubeVersion.APIVersions }}
	
	
	
.Template
	labels:
		filepath: {{ .Template.Name }}
		basepath: {{ .Template.BasePath }}

		
	
	渲染的结果：
	labels:
		filepath: mychart/templates/deployment.yaml
		basepath: mychart/templates

	
	
	
二、函数与管道
	用法：
		{{ 函数名 参数1 参数2 }}
		{{ 值 | 函数名 }}
		
	
	# 不支持管道的用法
	{{ template "mychart.name" . }}
	
	
	# 加引号
	basepath: {{ quote .Template.BasePath }}
    x: {{ .Release.IsUpgrade | quote}}

	
	# 管道详细用法
	food: {{ .Values.favorite.food | upper | repeat 3 | quote }}
	
	
	# default设置默认值
	
	food: {{ .Values.favorite.food | default "rice" | upper | quote }}
	
	
	示例：
	 food: {{ .Values.favorite.food | default (printf "%s-egon" (include "mychart.name" .)) }}
	
	
	
三、if判断

语法：

{{ if 条件1 }}
  # Do something
{{ else if 条件2 }}
  # Do something else
{{ else }}
  # Default case
{{ end }}


示例：使用 {{- 可以清除语法占用的空行
{{- if .Values.db.debugMode | not}}
    name: DB_DEBUG1
    value: "false"
{{- else }}
    name: DB_DEBUG2
    value: "true"
{{- end }}

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}
  
 
判断条件也可以是比较运算（eq、ne、lt、gt）

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}
```



判断条件里也可以引入and、or ，也可以用括号`（）`进行分割。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- if and (eq .Values.favorite.drink "coffee") (gt (int .Values.replicaCount) 3) }}
  mug: true
  {{- end }}
```


apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- if and (eq .Values.favorite.drink "coffee") (gt (int .Values.replicaCount) 3) }}
  mug: true
  {{- end }}
  

空格与空行的控制
  
  {{- if and (eq .Values.favorite.drink "coffee") (gt (int .Values.replicaCount) 3)}}
  mug: true
  {{- end }}

  
  模版语法渲染之后遗留的换行符用-去掉
  内容部分的缩进空格，可以自己人为控制缩进空格数，
  
  

  # indent往右缩进n个空格
  # nindent往左缩进n个空格
  
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  myvalue: "Hello World"
	  drink: {{ .Values.favorite.drink | default "tea" | quote }}
	  food: {{ .Values.favorite.food |default "abc" | upper | quote }}
	  {{- if and (eq .Values.favorite.drink "coffee") (gt (int .Values.replicaCount) 3)}}
		{{ nindent 2 "mug: true" }}
	  {{- end }}
	  xxx: aaaa
	  yyy: bbbb



with声明一段作用域：简写方便
	# values.yaml
	db:
	  debugMode: true
	  dbinfo:
		user: egon
		pass: "123"
		port: 3306


	# yaml清单
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  {{- with .Values.db.dbinfo }}
	  user: {{ .user }}
	  pass: {{ .pass }}
	  port: {{ .port }}
	  {{- end }}
	  release: {{ .Release.Name }}  # 内置对象的访问要从with内拿出来      
	
	
	
	
range语法：


	# values.yaml
	favorite:
	  drink: coffee
	  food: pizza
	names:
	  - egon
	  - lili
	  - jack
	  - tom

	# yaml清单：
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  myvalue: "Hello World"

	  nameList1: |-
		{{- $relname := .Release.Name -}}
		{{- range .Values.names }}
		- "{{ . | title }}-{{ $relname }}"
		{{- end }} 
		
	  nameList2: |-
		{{- range tuple "EGON1" "EGON2" "EGON3" }} # 直接声明一个数组来遍历
		- {{ . }}-{{ $relname }}
		{{- end }}

	  nameList3: |-
		{{- range .Values.favorite }}
		- {{ . | quote }}
		{{- end }}

	  nameList4: |-
		{{- range $key,$value := .Values.favorite }}
		{{ $key }}: {{ $value | quote }}
		{{- end }}


	  nameList5: |-
		{{- range $index,$value := .Values.names }}
		{{ $index }}: {{ $value | quote }}
		{{- end }}



变量定义：


	$. 永远指向全局
	
	[root@k8s-master-01 /test1/mychart]# cat templates/cm.yaml 
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  myvalue: "Hello World"

	  {{- with .Values.db.dbinfo }}
	  user: {{ .user }}
	  pass: {{ .pass }}
	  port: {{ .port }}
	  release: {{ $.Release.Name }}
	  {{- end }}

	  nameList1: |-
		{{- range .Values.names }}
		- "{{ . | title }}-{{ $.Release.Name }}"
		{{- end }} 
		
	  nameList2: |-
		{{- range tuple "EGON1" "EGON2" "EGON3" }} # 直接声明一个数组来遍历
		- {{ . }}-{{ $.Release.Name }}
		{{- end }}

	  nameList3: |-
		{{- range .Values.favorite }}
		- {{ . | quote }}
		{{- end }}

	  nameList4: |-
		{{- range $key,$value := .Values.favorite }}
		{{ $key }}: {{ $value | quote }}
		{{- end }}
	  nameList5: |-
		{{- range $index,$value := .Values.names }}
		{{ $index }}: {{ $value | quote }}
		{{- end }}

	

	
	
模版全局命令文件：

	示例1：
	[root@k8s-master-01 /test1]# cat mychart/templates/_helpers.tpl 
	{{- define "mychart.labels" }}
	  labels:
		generator: helm
		date: {{ now | htmlDate }}
	{{- end }}

	
	[root@k8s-master-01 /test1/mychart]# cat templates/cm.yaml 
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	  {{- template "mychart.labels"  }}
	  
	data:
	  myvalue: "Hello World"

		
	
	
	#  示例2：需要传入作用域
	[root@k8s-master-01 /test1]# cat mychart/templates/_helpers.tpl 
	{{- define "mychart.labels" }}
	  labels:
		generator: helm
		date: {{ now | htmlDate }}
		chart: {{ .Chart.Name }}
		version: {{ .Chart.Version }}
	{{- end }}

	[root@k8s-master-01 /test1/mychart]# cat templates/cm.yaml 
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	  {{- template "mychart.labels" . }}
	  
	data:
	  myvalue: "Hello World"

	
	注意：template不能与管道符号联用，此时可以用include（支持管道符）
	[root@k8s-master-01 /test1]# cat mychart/templates/_helpers.tpl 
	{{- define "mychart.labels" }}
	generator: helm
	date: {{ now | htmlDate }}
	chart: {{ .Chart.Name }}
	version: {{ .Chart.Version }}
	{{- end }}

	
	
	[root@k8s-master-01 /test1/mychart]# cat templates/cm.yaml 
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	  labels:
	  {{- include "mychart.labels" . | indent 4 }}
	  
	data:
	  myvalue: "Hello World"

	
	
	
	
	
访问/引入外部普通文件
	
	有时候需要导入一个不是模板的文件并注入其内容，而不通过模板渲染器获得内容。

	Helm 提供了一个 `.Files` 对象对文件的访问，但是在模板中使用这个对象之前，还有几个需要注意的事项值得一提：

	- 可以在 Helm chart 中添加额外的文件，这些文件也会被打包，不过需要注意，由于 Kubernetes 对象的存储限制，Charts 必须小于 1M

	  ```shell
	  ========================= 一个关于helm install的大坑
	  
	  注意注意注意，chart包目录下不要放无用的大文件，遇到过一个情况就是把镜像文件也放到了chart目录下，结果导致helm install阻塞在原地半天报错：hcreate: failed to create: Request entity too large: limit is 3145728
	  [root@jsswx191 mongodb-vpr]# pwd
	  /root/acg-vpr/k8s-service/mongodb-vpr
	  [root@jsswx191 mongodb-vpr]# ls
	  Chart.yaml  install.sh  k8s-mongo-sidecar.tar  mongo.tar  templates  uninstall.sh  upgrade.sh  values.yaml
	  ```

	- 出于安全考虑，通过 `.Files` 对象无法访问某些文件 ，例如

	  - 无法访问 `templates/` 下面的文件
	  - 无法访问使用 `.helmignore` 排除的文件
	
	
	
	
	准备工作：
	cd mychart/

	```text
	# 1、config1.toml
	cat > config1.toml << EOF
	message = Hello from config 1
	EOF

	# 2、config2.toml
	cat > config2.toml << EOF
	message = This is config 2
	EOF

	# 3、config3.toml
	cat > config3.toml << EOF
	message = Goodbye from config 3
	EOF
	```

	目录结构

	```
	[root@k8s-master-01 /test1/mychart]# tree  .
	.
	├── charts
	├── Chart.yaml
	├── config1.toml
	├── config2.toml
	├── config3.toml
	├── templates
	│   └── test-cm.yaml
	└── values.yaml

	```


	
	示例1：
	
	[root@k8s-master-01 /test1/mychart]# cat templates/cm.yaml 
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  config1.toml: |-
		{{ .Files.Get "config1.toml" }}
	  config2.toml: |-
		{{ .Files.Get "config2.toml" }}
	  config3.toml: |-
		{{ .Files.Get "config3.toml" }}

	
	
	示例2：
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
	  {{ . }}: |-
		{{ $.Files.Get . | indent 4}}
	  {{- end }}

	  
	示例3： 对内进行base64转码
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: {{ .Release.Name }}-configmap
	data:
	  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
	  {{ . }}: |-
		{{ $.Files.Get . | b64enc | indent 4 }}
	  {{- end }}
	~                         
		
	
	
	
	示例4： Lines
	cd /test1/mychart
	
	mkdir foo
	cat > foo/bar.txt << EOF
1111
2222
3333
EOF



[root@k8s-master-01 /test1/mychart]# cat foo/bar.txt 
1111
2222
3333


	# 模版文件编写方式1
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  some-file.txt: |
{{ .Files.Get "foo/bar.txt" | indent 4}}
                                               
	
	
	
	# 模版文件编写方式2
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  some-file.txt: |
  {{- range .Files.Lines "foo/bar.txt" }}
    {{ . }}
  {{- end }}                                  
	
	
	
	
	
	

NOTES.txt文件使用：

	[root@k8s-master-01 /test1/mychart]# cat /test1/mychart/templates/NOTES.txt 
	Thank you for installing {{ .Chart.Name }}.

	Your release is named {{ .Release.Name }}.

	To learn more about the release, try:

	  $ helm status {{ .Release.Name }}
	  $ helm get {{ .Release.Name }}

	
	使用示例：
	[root@k8s-master-01 /test1/mychart]# helm install xxx . 
	NAME: xxx
	LAST DEPLOYED: Sat Aug 31 11:56:53 2024
	NAMESPACE: default
	STATUS: deployed
	REVISION: 1
	TEST SUITE: None
	NOTES:
	Thank you for installing mychart.

	Your release is named xxx.

	To learn more about the release, try:

	  $ helm status xxx
	  $ helm get xxx


~~~

