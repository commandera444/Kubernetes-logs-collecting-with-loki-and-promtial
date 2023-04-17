# Add Datasource for Grafana.

Navigate in DataSource Configuration Section.

![Sources](../../Screens/datasourcebutton.png)

Add new DataSource.

![Sources](../../Screens/addbutton.png)

Select Loki.

![Sources](../../Screens/lokiicon.png)

* In URL Filed You must add loki url

```bash
http://loki-loki-distributed-gateway.<YOUR NAMESPACE>.svc.cluster.local
```

![Sources](../../Screens/url.png)

Save and test.

![Sources](../../Screens/testdatasource.png)