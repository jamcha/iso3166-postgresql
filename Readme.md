# iso3166

[ISO3166](https://es.wikipedia.org/wiki/ISO_3166) es un estándar internacional para códigos de paises y estados (sub-divisiones).

En este proyecto se intenta obtener la información estructurada en tablas de [PostgreSQL](https://www.postgresql.org/)

La idea principal es obtener esta información en español, aunque se puede cambiar a otros idiomas. Afortunadamente existe un
[proyecto](https://github.com/esosedi/3166) que proporciona esta información en archivos **json**. Esa será nuestra fuente de datos
principal. Como comentan ellos *"This is the best source for iso3166 codes you can found"* y a su vez obtienen la información
de otros medios. ¡Gracias!

Podrás encontrar ["un dump"](https://github.com/jamcha/iso3166-postgresql/tree/master/dump) de la base de datos sin comprimir con los paises y estados en español. Adicionalmente otra con
datos sobre colonias y municipios de México.

Se ha documentado el [procedimiento completo](Procedimiento.md) usado para crear todas las tablas. Como extra, también se documentó el 
[proceso para México](Mexico.md) donde se cargan colonias y municipios.

