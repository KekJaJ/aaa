# Habilitación de Operador grafana monitoreo de caches

## Despliegue del Operator

### Pre-requisitos

1. Poseer un client OC 
2. Acceso a openshift con usuario con rol cluster-admin
3. Namespace con data grid y namespace distinto con grafana version alpha 
4. Operador Grafana version alpha v4.4.1 habilitado en namespace (namespace donde se administrara el monitoreo)

### 1. Habilitacion de monitoreo en clusteres de caches

1. Login sobre Openshift
```bash
oc login http://....:port
```
2. Ingresar en proyecto el cual posea los caches a monitorear
```bash
oc project <namespace>
```
3. Añadir anotacion al a los cluster infinispan CR *infinispan.org/monitoring* en *true* como se muestra a continuacion para habilitar el monitoreo de sus
infinipan CR
```bash
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan
  annotations:
    **infinispan.org/monitoring: 'true'**
```
4. Se creará un *serviceMonitor* llamado (namespace)-monitoring :
```bash
NameSpace: poc-redhat-babysitting
Tipo: ServiceMonitor
Nombre en OCP: sso-infinispan-monitor
Nombre archivo: sso-infinispan-monitor.yaml
```
5. Ingresar a Openshift web console y como administrador, nos dirigimos a nuestro panel, abrimos la pestaña *observe/metrics* confirmando que podemos buscar la siguiente metrica

```bash
vendor_cache_manager_default_cluster_size
```
### 2. Habilitacion se user-defined projects 

#### Este es un pre-requesito para el debido monitoreo con grafana antes de su configuracion

1. Editamos el *ConfigMap*  llamado cluster-monitoring-config
```bash
oc -n openshift-monitoring edit configmap cluster-monitoring-config
``` 
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
2. Creaamos un serviceAccount para permitir a grafana leer metricas de data grid
```bash
NameSpace: grafana-monitoring
Tipo: serviceAccount
Nombre en OCP: infinispan-monitoring
Nombre archivo: serviceAccount.yaml
``` 
```bash
oc apply -f service-account.yaml
``` 
> Otorgamos permisos *cluster-monitoring-view* al *ServiceAccount*
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
        **eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1LZU...**
      type: prometheus
      url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
```
>aplicamos el source 
```bash
oc apply -f grafana-datasource.yaml
```
### 4. Configuracion de paneles para desplegar en grafana UI
1. creamos un ConfigMap en el operador donde se encuentra nuestros caches de data grid 
```bash
NameSpace: poc-redhat-babysitting
Tipo: ConfigMap
Nombre en OCP: infinispan-operator-config
Nombre archivo: configmap-operator-infinispan.yaml
``` 
```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: infinispan-operator-config
data:
  grafana.dashboard.monitoring.key: middleware
  grafana.dashboard.name: infinispan
  grafana.dashboard.namespace: infinispan
``` 
***monitorin.key: es la clave de supervision***


***dashborad.name: es el nombre del grafana dashboard que se creará en nuestro operador de grafana***


***dashboard.namespace: es el nombre de nuestro namespace donde se encuentra nuestro cache***



2. creamos o actualizamos el *infinispan-operator-config* 
```bash
oc apply -f infinispan-operator-config.yaml
```   
2. le otorgamos permisos de administrador a nuestro serviceAccount creado con anterioridad
```bash
oc policy add-role-to-user edit system:serviceaccount:poc-redhat-babysitting:infinispan-operator-controller-manager -n grafana-monitoring
```  

3. obtenemos la ruta url donde se encuentra nuestro Grafana disponible
```bash
oc obtener rutas grafana-ruta -o jsonpath=https://"{.spec.host}"
```  