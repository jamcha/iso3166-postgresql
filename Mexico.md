# México

Para México, [SEPOMEX](https://www.correosdemexico.gob.mx) (Servicio Postal Mexicano) ofrece datos de todos los códigos
postales actualizados en un solo archivo que incluye Estados y Municipios (ciudades). De ahí se obtienen los catálogos
de Municipios y de Colonias con su código postal.

# Procedimiento

1. Descargar fuente de datos [Sepomex](https://www.correosdemexico.gob.mx/SSLServicios/ConsultaCP/CodigoPostal_Exportar.aspx) en
   formato TXT. Descomprimir archivo y copiarlo a /tmp

2. Cargar base de datos

```bash
psql iso3166
```

3. Cargar datos a una tabla temporal. El archivo original trae los HEADERS en la línea 2 y
   un "mensaje" en la línea 1...crearemos todos los campos TEXT para hacer todo el proceso
   desde SQL.

```sql
CREATE TEMP TABLE sepomex_tmp 
  (d_codigo         TEXT,
   d_asenta         TEXT,
   d_tipo_asenta    TEXT,
   D_mnpio          TEXT,
   d_estado         TEXT,
   d_ciudad         TEXT,
   d_CP             TEXT,
   c_estado         TEXT,
   c_oficina        TEXT,
   c_CP             TEXT,
   c_tipo_asenta    TEXT,
   c_mnpio          TEXT,
   id_asenta_cpcons TEXT,
   d_zona           TEXT,
   c_cve_ciudad     TEXT);
COPY sepomex_tmp FROM '/tmp/CPdescarga.txt' CSV DELIMITER '|' HEADER ENCODING 'LATIN9';
DELETE FROM sepomex_tmp WHERE d_codigo='d_codigo';
```

4. Tabla municipios
```sql
ALTER TABLE sepomex_tmp ADD idestado INTEGER;
UPDATE sepomex_tmp SET idestado = tmp.idestado
  FROM (SELECT DISTINCT c_estado,d_estado,e.idestado
          FROM sepomex_tmp
          LEFT JOIN estados AS e ON (idpais=484 AND nombre=d_estado)) AS tmp
 WHERE sepomex_tmp.c_estado = tmp.c_estado;
CREATE TABLE municipios (
  idmunicipio   SERIAL PRIMARY KEY,
  nombre        TEXT,
  refs          JSONB,
  idestado      INTEGER REFERENCES estados
);
INSERT INTO municipios (nombre,idestado,refs)
            (SELECT d_mnpio,idestado,json_build_object('sepomex',refs::JSON)
               FROM (SELECT DISTINCT d_mnpio,idestado,json_build_object('idmunicipio',c_mnpio)::TEXT AS refs
                       FROM sepomex_tmp
                      ORDER BY idestado,d_mnpio) AS x);
-- Ajustar la secuentcia.
-- Tal vez sea buena idea "saber" que los nuevos municipios agregados (si los hay) serán del 5,000 en adelante
ALTER SEQUENCE municipios_idmunicipio_seq RESTART 5000;
-- o si no
-- SELECT setval('municipios_idmunicipio_seq',(SELECT max(idmunicipio) FROM municipios),TRUE);
```

5. Tabla final de colonias

```sql
CREATE TABLE colonias (
  idcolonia     SERIAL PRIMARY KEY,
  nombre        TEXT,
  cp            INTEGER,
  refs          JSONB,
  idmunicipio   INTEGER REFERENCES municipios
);
INSERT INTO colonias (nombre,cp,idmunicipio,refs)
                     (SELECT d_asenta,d_codigo::INTEGER,idmunicipio,
                             json_build_object('codigo',         d_codigo,
                                               'tipo_asenta',    d_tipo_asenta,
                                               'tipo_asenta_id', c_tipo_asenta,
                                               'oficina',        c_oficina,
                                               'zona',           d_zona,
                                               'ciudad',         d_ciudad,
                                               'ciudad_id',      c_cve_ciudad)
                        FROM sepomex_tmp AS s
                        LEFT JOIN municipios AS m ON (m.refs->'sepomex'->>'idmunicipio'=s.c_mnpio AND
                                                      m.idestado=s.idestado)
                       ORDER BY d_estado,d_mnpio,d_asenta);
-- Ajustar la secuentcia.
-- Tal vez sea buena idea "saber" que las nuevas colonias agregados (si los hay) sean del 20,000 en adelante
ALTER SEQUENCE colonias_idcolonia_seq RESTART 20000;
-- o si no
-- SELECT setval('colonias_idcolonia_seq',(SELECT max(idcolonia) FROM colonias),TRUE);


6. Prueba

```sql
SELECT c.idcolonia, c.nombre AS colonia, c.refs->>'tipo_asenta' AS tipo_asentamiento,
       c.cp, m.nombre AS municipio, e.nombre AS estado
  FROM colonias AS c
  LEFT JOIN municipios AS m USING (idmunicipio)
  LEFT JOIN estados AS e USING (idestado)
 WHERE cp = 64180;
```