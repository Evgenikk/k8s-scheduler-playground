В этой статье постараюсь на базовом уровне пояснить как работают taints и tolerations в kubernetes. 


### Taints and tolerations

Назначить  taint на ноду можно при помощи `kubectl taint nodes node01 key=value:effect`, убрать  taint можно командой `kubectl taint nodes node01 key=value:effect-`

Здесь вместо effect можно использовать:

* `NoSchedule` тогда планировщик не будет размещать поды на ноду с  taint, но уже запущеные на ноде поды продолжат работу на ней
* `PreferNoSchedule`  по-возможности kubernetes будет не назначать поды на ноду с  taint
* `NoExecute` поды будут убраны с ноды

`key=value` позволяет в дальнейшем добавить в  спецификацию пода `tolerations`, по которому будут выбраны поды, на которых *не будет распространяться* действие `taint`.

Давайте рассмотрим на примере. Предположим у нас есть 4 

### Affinity

Если `taints` нужны для того, чтобы планировщик НЕ назначал поды на некоторые ноды, то `affinity` позволяют управлять тем, куда планировщик *назначит* конкретный под. Можно выделить три типа affinity:

* nodeAffinity - позволяет назначить под на ноды попадающие под условия
* podAffinity  - позволяет назначить под на ноды, где уже есть поды попадающие под условия
* podAntiAffinity - запрещает назначать под на ноды, где есть поды попадающие под условия


У affinity  есть два варианта воздействия preferredDuringSchedulingIgnoredDuringExecution  и requiredDuringSchedulingIgnoredDuringExecution. Как видно в обоих случаях  affinity игнорируется, если под уже запущен, как вариант решения удалять поды, которые не попадают под affinity или сделать drain node

### Разберемся на примере

Предположим, у нас есть ноды двух типов, маленькие и большие. В таком случае, мы хотим избежать, чтобы большие ноды были загружены подами, которым не требуетя много ресурсов.  В таком случае на ноды с большим кол-вом ресурсов можно повестить taint, запрещаюший запускать на ней поды. 

```
kubectl taint node largeNode size=large:NoExecute 
```

Теперь, если мы начнем создавать поды в кластере, они не будут запускаться на largeNode `kubectl get pods -o wide`:
```
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE                 NOMINATED NODE   READINESS GATES
workload1-7dbdf759dc-dm8tf   1/1     Running   0          10m   10.24.2.9   smallnode1           <none>           <none>
workload1-7dbdf759dc-mckpr   1/1     Running   0          27m   10.24.2.8   smallnode1           <none>           <none>
workload1-7dbdf759dc-rtbd5   1/1     Running   0          27m   10.24.1.4   mediumnode1          <none>           <none>
```

Сейчас добавим немного подов, которым дадим tolerations, позволяющий назначать эти поды на largenode, для этого в спецификации пода в deployment укажем:

```
      tolerations:
        - key: size
          operator: Equal
          value: large
          effect: NoExecute
```

Наблюдаем результат `kubectl get pods -o wide`:
```
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
largepods-564b99d8-57fhb     1/1     Running   0          7s    10.24.0.5   largeNode        <none>           <none>
largepods-564b99d8-66gbz     1/1     Running   0          7s    10.24.0.4   largeNode        <none>           <none>
largepods-564b99d8-nwb6f     1/1     Running   0          7s    10.24.1.6   smallnode1       <none>           <none>
workload1-7dbdf759dc-dm8tf   1/1     Running   0          10m   10.24.2.9   mediumnode1      <none>           <none>
workload1-7dbdf759dc-mckpr   1/1     Running   0          27m   10.24.2.8   mediumnode1      <none>           <none>
workload1-7dbdf759dc-rtbd5   1/1     Running   0          27m   10.24.1.4   smallnode1       <none>           <none>
```

Как видим, один из подов запустился на маленькой ноде, чтобы этого избежать используем nodeAffinity. Для начала назначим на largeNode метку:
```
kubectl label node lageNode  size=large
kubectl label node mediumNode  size=medium
```

Далее добавляем affinity в спеуификацию пода в deployment:
```
      affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: size
                  operator: In
                  values:
                  - large
                  - medium
```

Таким образом поды будут попадать только на ноды, на которых метка  size принимает или значение large или значение medium:

```
NAME                         READY   STATUS        RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
largepods-564b99d8-qwwwg     1/1     Terminating   0          92s   10.24.2.10   lageNode      <none>           <none>
largepods-689b86fc84-r8s6p   1/1     Running       0          92s   10.24.0.7    mediumnode1   <none>           <none>
largepods-689b86fc84-rrnsv   1/1     Running       0          92s   10.24.2.11   lageNode      <none>           <none>
largepods-689b86fc84-s2n2r   1/1     Running       0          92s   10.24.0.6    mediumnode1   <none>           <none>
largepods-689b86fc84-t8jkj   1/1     Running       0          90s   10.24.0.8    mediumnode1   <none>           <none>
largepods-689b86fc84-xv2rf   1/1     Running       0          90s   10.24.0.9    mediumnode1   <none>           <none>
```


