#############################
### Instalación de PoP Monitoreo Sintético ###
#############################


<img width="1470" height="530" alt="image" src="https://github.com/user-attachments/assets/3f3cc775-521c-44cd-a9b8-bcfe6a212633" />


Usar como nombre del POP el número de usuario asignado (PoPUserXX)

```
helm upgrade --install synthetic-pop \
	--repo "https://agents.instana.io/helm" \
	--namespace nspop \
	--create-namespace \
	--set downloadKey="lYXhsTslRHacNL7TY_EXDQ" \
	--set controller.location="PoPUser00;PoPUser00;Lima;PERU;0;0;Mainsoft" \
	--set controller.clusterName="PoPUser00" \
	--set controller.instanaKey="lYXhsTslRHacNL7TY_EXDQ" \
	--set controller.instanaSyntheticEndpoint="https://lab.mainsoft.com/synthetics" \
	--set redis.tls.enabled=false \
	--set redis.password="a1fc5d01bcbb" \
	synthetic-pop


kubectl patch deployment synthetic-pop-controller -n nspop --type='json' -p='[
  {
    "op": "add", 
    "path": "/spec/template/spec/hostAliases", 
    "value": [
      {
        "ip": "10.249.128.36", 
        "hostnames": ["lab.mainsoft.com"]
      }
    ]
  }
]'
```

<img width="904" height="744" alt="image" src="https://github.com/user-attachments/assets/25985a47-5754-426d-915f-9ed0a802b926" />



Esperar que todos los pods levanten (tarda en levantar 10 min)
```
watch kubectl get pods -n nspop
```

<img width="747" height="129" alt="image" src="https://github.com/user-attachments/assets/5cc3c499-952b-4b69-919a-951ebee1373e" />

Validar que el PoP figure en Instana:

<img width="1726" height="457" alt="image" src="https://github.com/user-attachments/assets/a3fcef34-e9b5-4a85-aac8-9096822f0cb2" />


Crear monitoreo sintético hacia el robotshop instalado

<img width="1231" height="393" alt="image" src="https://github.com/user-attachments/assets/5888b68d-c1a6-4ff5-bc52-ca1a7a9627aa" />

Seleccionar el tipo de monitoreo sintético

<img width="787" height="646" alt="image" src="https://github.com/user-attachments/assets/d22af350-29d2-4889-b1dd-ca8930d90c52" />

Colocar la URL de la aplicación robotshop

<img width="994" height="646" alt="image" src="https://github.com/user-attachments/assets/48dd82e1-224d-451d-95d8-6ff798666c21" />

Seleccionar el PoP instalado

<img width="969" height="376" alt="image" src="https://github.com/user-attachments/assets/27e72006-3d99-4d41-a177-6d8917f0837c" />

Ajustar la frecuencia y colocar el nombre del monitoreo sintético

<img width="981" height="640" alt="image" src="https://github.com/user-attachments/assets/3be71c51-37a9-4774-b70d-e8ea7e6ecf99" />

Asociar monitoreo sintético a una perspectiva de aplicación y website

<img width="986" height="643" alt="image" src="https://github.com/user-attachments/assets/a5d5b053-7ab2-48ee-8b38-a7b2c2b5222f" />

Asociar un Team y crear el monitoreo sintético

<img width="929" height="640" alt="image" src="https://github.com/user-attachments/assets/eac85d2b-7fab-4796-b52d-8512d42dbae7" />

Actualizar la página y validar el monitoreo sintético (

<img width="934" height="451" alt="image" src="https://github.com/user-attachments/assets/dc028377-727c-4f08-9927-c160b44605c9" />

Para validar la llegada a la URL destino desde el pod 

Nota: reemplazar el nombre del pop y la URL

```
kubectl get pods -n nspop
kubectl exec -it -n nspop synthetic-pop-browserscript-playback-engine-xxxxxxxxxx-xxxxxx -- curl -I http://10.249.128.38:8080/
```

<img width="1035" height="208" alt="image" src="https://github.com/user-attachments/assets/09d33874-f8f9-4315-a0fb-18051597d6c7" /><br>


Para desinstalar el pop

```
helm uninstall synthetic-pop -n nspop
```

Script para saber el request/limit/used de cpu y memoria en los pods
```
cat > k8s_resource_audit.sh << 'OUTER_EOF'
#!/bin/bash

# Comprobar si jq está instalado
if ! command -v jq &> /dev/null; then
    echo "Error: 'jq' no está instalado. Instálalo con: sudo dnf install jq"
    exit 1
fi

printf "%-30s %-20s %-10s %-10s %-10s %-10s %-10s %-10s\n" "POD" "CONTAINER" "CPU_REQ" "CPU_LIM" "CPU_USE" "MEM_REQ" "MEM_LIM" "MEM_USE"
echo "---------------------------------------------------------------------------------------------------------------------------------------"

# Obtener métricas actuales (Usage)
temp_metrics=$(kubectl top pod -A --containers --no-headers)

# Obtener Specs (Requests/Limits)
kubectl get pods -A -o json | jq -r '
  .items[] | .metadata.namespace as $ns | .metadata.name as $pod | .spec.containers[] | .name as $cont |
  [$ns, $pod, $cont, 
   (.resources.requests.cpu // "0"), (.resources.limits.cpu // "0"), 
   (.resources.requests.memory // "0"), (.resources.limits.memory // "0")] | @tsv' | \
while IFS=$'\t' read -r ns pod cont cpu_req cpu_lim mem_req mem_lim; do
    
    # Buscar el uso actual en el dump de metrics
    usage_line=$(echo "$temp_metrics" | grep "$ns" | grep "$pod" | grep "$cont")
    cpu_use=$(echo "$usage_line" | awk '{print $4}')
    mem_use=$(echo "$usage_line" | awk '{print $5}')

    # Formatear salida
    printf "%-30s %-20s %-10s %-10s %-10s %-10s %-10s %-10s\n" \
        "${ns}/${pod:0:20}..." "$cont" "$cpu_req" "$cpu_lim" "${cpu_use:-N/A}" "$mem_req" "$mem_lim" "${mem_use:-N/A}"
done
echo "---------------------------------------------------------------------------------------------------------------------------------------"
kubectl describe node | grep -A 15 "Allocated resources"

OUTER_EOF

chmod +x k8s_resource_audit.sh
./k8s_resource_audit.sh
```
