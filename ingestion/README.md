# ingestion/
Scripts que **descargan datos de las fuentes externas** (FIRMS, EUMETSAT, AEMET, OSM) y los suben al volumen de Databricks.
Corren en **GitHub Actions**, no en Databricks, porque Databricks Free Edition limita la salida a internet.
