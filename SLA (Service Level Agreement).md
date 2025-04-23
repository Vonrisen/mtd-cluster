# SLA (Service Level Agreement)

## SLA per il Database

### 1. Disponibilità del Database
- **SLA**: 99.9% di uptime mensile
- **Metrica**: `mysql_up{service="mysql-exporter-prometheus-mysql-exporter"}`
- **Soglia**: Alert se il valore scende sotto la soglia prevista

### 2. Connessioni Attive
- **SLA**: < 80% del limite massimo (`max_connections`)
- **Metrica**:
  ```
  sum(max_over_time(mysql_global_status_threads_connected{job=~"$job", instance=~"$instance"}[$__interval]))
  ```
- **Soglia**: Alert se supera l'80% del limite massimo

## SLA per il Backend

### 1. Disponibilità del Servizio
- **SLA**: 99.5% di uptime mensile (massimo 3.6 ore di downtime al mese)
- **Metrica**:
  ```
  avg_over_time(up{service="backend-service"}[1h]) * 100
  ```
- **Soglia**: Alert se il valore scende sotto la soglia prevista

### 2. Percentuale di successo delle richieste HTTP
- **SLA**: ≥ 95% delle richieste
- **Metrica**:
  ```
  100 * (
    sum(rate(flask_http_request_total{service="backend-service", status=~"200"}[1h])) /
    sum(rate(flask_http_request_total{service="backend-service"}[1h]))
  )
  ```
- **Soglia**: Alert se < 95%

### 3. Errori HTTP
- **SLA**: < 20 errori all'ora
- **Metrica**:
  ```
  sum(rate(flask_http_request_duration_seconds_count{status!="200"}[30s]))
  ```
- **Soglia**: Alert se supera i 20 errori all'ora

### 4. Affidabilità del Backend
- **SLA**: ≥ 95% di tasso di affidabilità mensile
- **Metrica**:
  ```
  100 * (
    sum(rate(flask_http_request_created{service="backend-service", status="200"}[5m])) /
    sum(rate(flask_http_request_created{service="backend-service"}[5m]))
  )
  ```
- **Soglia**: Alert se < 95%

## SLA per il Frontend (Nginx)

### 1. Disponibilità del Frontend
- **SLA**: 99.7% di uptime mensile (massimo 2 ore di downtime al mese)
- **Metrica**:
  ```
  avg_over_time(nginx_up{service="frontend-service"}[1h])
  ```
- **Soglia**: Alert se il valore scende sotto la soglia prevista

## SLA per il Cluster Kubernetes

### 1. Disponibilità dei Nodi
- **SLA**: 99.9% di uptime per nodo (massimo 43 minuti di downtime al mese)
- **Metrica**:
  ```
  avg_over_time(kube_node_status_condition{condition="Ready", status="true"}[15m])
  ```
- **Soglia**: Alert se il valore scende sotto la soglia prevista

### 2. Utilizzo CPU
- **SLA**: < 80% per nodo
- **Metrica**:
  ```
  100 - (avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  ```
- **Soglia**: Alert se > 80% per più di 15 minuti

### 3. Utilizzo Memoria
- **SLA**: < 80% per nodo
- **Metrica**:
  ```
  100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
  ```
- **Soglia**: Alert se > 80%

