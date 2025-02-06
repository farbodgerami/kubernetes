modify new .env(new svc for database addresses)

to create configmaps:
```
kubectl create configmap PROJECT_NAME_1_configmap --from-env-file=./.env_PROJECT_NAME_1_stage
kubectl create configmap pay_configmap --from-env-file=./.env_pay_stage
kubectl create configmap DOMAIN_NAME_1front_configmap --from-env-file=./.env_front_stage
kubectl create configmap backoffice_configmap --from-env-file=./.env_backoffice_stage
```
