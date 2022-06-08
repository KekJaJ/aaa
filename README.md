# Monitoring Data Grid services - Guide - Red Hat Data Grid 8.3

Link referencia: https://access.redhat.com/documentation/en-us/red_hat_data_grid/8.3/guide/990f6af5-2a9c-4af1-afaf-d2ea562c5bf8

### Pre-requisitos

1. Poseer un client OC 
2. Acceso a openshift con usuario con rol cluster-admin
3. Namespace con data grid y namespace con grafana version alpha v4.4.1 donde se realizará el monitoreo.

### 1. Habilitacion de monitoreo en namespace DataGrid

1. Login sobre Openshift
```bash
oc login http://....:port
```
2. Ingresar en proyecto el contiene el operador de data grid
```bash
oc project <namespace> 
```
3. Cuando se generá la habilitación de Data Grid a traves de operador se genera un CR al cual se le debe añadir anotacion al a los cluster infinispan CR *infinispan.org/monitoring* en *true* como se muestra a continuacion para habilitar el monitoreo de sus
infinipan CR
```bash
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan
  annotations:
    **infinispan.org/monitoring: 'true'**
```
4. Al ingresar esta anotación se genera automaticamente se creará un *serviceMonitor* sobre el mismo namespace:
```bash
NameSpace: poc-redhat-babysitting
Tipo: ServiceMonitor
Nombre en OCP: sso-infinispan-monitor
```
5. Para confirmar que el monitoreo este activo se debe ir Metrics y validarlo con la query:
Ingresar a Openshift web console y como administrador, nos dirigimos a nuestro panel, abrimos la pestaña *observe/metrics* confirmando que podemos buscar la siguiente metrica
```bash
vendor_cache_manager_default_cluster_size
```
### 2. Habilitacion se user-defined projects 

#### Este es un pre-requesito para el debido monitoreo con grafana antes de su configuracion

1. Editamos el *ConfigMap*  llamado cluster-monitoring-config dentro del namespace openshift-monitoring
```bash
oc -n openshift-monitoring edit configmap cluster-monitoring-config
``` 
## *Si esta etiqueya ya se encuentra en true no se recomienda reiniciar esta (true a false y nuevamente true)*

> Añadimos la variable *enableUserWorkload* en *true* a la escala de data/config.yaml
```bash
 apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    **enableUserWorkload: true**
```
### 3. Habilitacion de monitoreo con grafana 

1. Ingresar a Openshift web console y como administrador, nos dirigimos a *installed operators>grafana*
> dentro del operador grafana creamos un grafana CR en la seccion superior llamada "grafana"
```bash
NameSpace: grafana-monitoring
Tipo: Grafana
Nombre en OCP: grafana-monitoring-infinispan
Nombre archivo: grafanaCR.yaml
``` 
```bash
oc project grafana-monitoring
```
```bash
oc apply -f grafanaCR.yaml
``` 
2. Creamos un serviceAccount para permitir a grafana leer metricas de data grid sobre el namespace donde se habilitó la instancia de grafana
```bash
NameSpace: grafana-monitoring
Tipo: serviceAccount
Nombre en OCP: infinispan-monitoring
Nombre archivo: serviceAccount.yaml
``` 
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: infinispan-monitoring
``` 
>  Aplicar permisos al SA
```bash
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z infinispan-monitoring
``` 
3. Creamos una fuente de datos de grafana 
> 1. Primero obtenemos un token proveniente del ServiceAccount creado anteriormente y lo guardamos para utilizarlo en los proximos pasos
```bash
oc serviceaccounts get-token infinispan-monitoring
```
4. definimos un data-source de tipo grafana incluyeendo el *token* obtenido anteriormente y lo ingresamos en el archivo a escala de *spec.datasources.secureJsonData.httpHeaderValue1*
```bash
NameSpace: grafana-monitoring
Tipo: GrafanaDataSource
Nombre en OCP: infinispan-grafana-datasource
Nombre archivo: grafana_dataSource.yaml
``` 
```bash
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: infinispan-grafana-datasource
spec:
  name: datasource.yaml
  datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        httpHeaderName1: Authorization
        timeInterval: 5s
        tlsSkipVerify: true
      name: Prometheus
    **secureJsonData:**
        httpHeaderValue1: >-
          Bearer
        **eyJhbGciOiJSUzI1NiIsImtp.....**
      type: prometheus
      url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
```
>aplicamos el source 
```bash
oc apply -f grafana-datasource.yaml
```
### 4. Generar GrafanaDashboard sobre Instancia de Grafana en diferente namespace:
1. creamos un ConfigMap en el operador donde se encuentra nuestros caches de data grid 
```bash
NameSpace: poc-redhat-babysitting
Tipo: ConfigMap
Nombre en OCP: infinispan-operator-config
Nombre archivo: configmap-operator-infinispan.yaml
``` 
2. Sobre el namespace de DataGrid se debe crear el siguiente configMap apuntando hacia el namespace de Grafana
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: infinispan-operator-config
data:
  grafana.dashboard.monitoring.key: middleware 
  grafana.dashboard.name: infinispan <--1
  grafana.dashboard.namespace: infinispan <--2
``` 
***monitorin.key: es la clave de supervision***

1- dashborad.name: es el nombre del grafana dashboard que se creará en nuestro operador de grafana

2- dashboard.namespace: es el nombre de nuestro namespace donde se encuentra nuestro cache


1. creamos o actualizamos el *infinispan-operator-config* 
```bash
oc apply -f infinispan-operator-config.yaml
```   
2. se le da permisos al operador del infinispan para realizar la creacion del folder y comunicacion con el namespace donde se encuentra el grafana
```bash
oc policy add-role-to-user edit system:serviceaccount:poc-redhat-babysitting:infinispan-operator-controller-manager -n grafana-monitoring
```  
3. obtenemos la ruta url donde se encuentra nuestro Grafana disponible
```bash
oc obtener rutas grafana-ruta -o jsonpath=https://"{.spec.host}"
```  
