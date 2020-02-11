---
title: rook-ceph debug
date: 2020-02-11 14:15:45
tags:
---
api/v1/minial 500
```
ceph dashboard ac-role-create admin-no-iscsi

for scope in dashboard-settings log rgw prometheus grafana nfs-ganesha manager hosts rbd-image config-opt rbd-mirroring cephfs user osd pool monitor; do
    ceph dashboard ac-role-add-scope-perms admin-no-iscsi ${scope} create delete read update;
done

ceph dashboard ac-user-set-roles admin admin-no-iscsi
```


#redis-cluster

kubectl -n redis exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 \
$(kubectl -n redis get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')