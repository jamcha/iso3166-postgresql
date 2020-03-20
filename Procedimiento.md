# Procedimiento

Se intenta hacer todo el procedimiento desde *PostgreSQL* (psql) para no depender de otras herramientas.

1. Descargar fuente de datos

```bash
wget https://raw.githubusercontent.com/esosedi/3166/master/i18n/dispute/UN/es.json -O /tmp/iso3166.json
```

> Selecciona el idioma (lenguaje) que desees. En este ejemplo es español

2. Cargar base de datos

```bash
createdb iso3166
psql iso3166
```

3. Cargar datos a una tabla temporal

```sql
CREATE TEMP TABLE iso3166_tmp (data JSONB);
COPY iso3166_tmp FROM '/tmp/iso3166.json';
CREATE TEMP TABLE iso3166_tmp_paises AS (SELECT * FROM jsonb_each((SELECT data FROM iso3166_tmp)));
```

4. Tabla final de paises
```sql
CREATE TABLE paises (
  idpais        SERIAL PRIMARY KEY,
  nombre        TEXT,
  iso           VARCHAR(2),
  iso3          VARCHAR(3),
  fips          VARCHAR(3),
  refs          JSONB
);
CREATE UNIQUE INDEX paises_idx_iso  ON paises (iso);
CREATE UNIQUE INDEX paises_idx_iso3 ON paises (iso3);
INSERT INTO paises (SELECT (value->>'numeric')::INTEGER AS idpais,
                            value->>'name'     AS name,
                            value->>'iso'      AS iso,
                            value->>'iso3'     AS iso3,
                            value->>'fips'     AS fips,
                            value->'reference' AS fererence
                      FROM iso3166_tmp_paises);
-- Ajustar la secuentcia.
-- Tal vez sea buena idea "saber" que los nuevos paises agregados (si los hay) serán del 1,000 en adelante
ALTER SEQUENCE paises_idpais_seq RESTART 1000;
-- o si no
-- SELECT setval('paises_idpais_seq',(SELECT max(idpais) FROM paises),TRUE);
```

5. Tabla final de estados. Usaremos una funcion para facilitar el proceso

```sql
CREATE OR REPLACE FUNCTION tmp_estados
  (OUT idpais     INTEGER,
   OUT name       TEXT,
   OUT iso        VARCHAR(3),
   OUT fips       TEXT,
   OUT admin      TEXT,
   OUT reference  JSONB)
  RETURNS SETOF RECORD AS $$
DECLARE
  r   RECORD;
  rr  RECORD;
BEGIN
  FOR r IN SELECT * FROM iso3166_tmp_paises LOOP
    idpais := (r.value->>'numeric')::INTEGER;
    FOR rr IN SELECT * 
                FROM jsonb_to_recordset(r.value->'regions')
                     AS X (iso       TEXT,
                           fips      TEXT,
                           name      TEXT,
                           admin     TEXT,
                           reference JSONB) LOOP
      iso       := rr.iso;
      fips      := rr.fips;
      name      := rr.name;
      admin     := rr.admin;
      reference := rr.reference;
      RETURN NEXT;
    END LOOP;
  END LOOP;
  RETURN;
END;$$
LANGUAGE plpgsql;
```

```sql
CREATE TABLE estados (
  idestado      SERIAL PRIMARY KEY,
  nombre        TEXT,
  iso           VARCHAR(3),
  fips          VARCHAR(8),
  admin         TEXT,
  refs          JSONB,
  idpais        INTEGER REFERENCES paises
);
-- CREATE UNIQUE INDEX estados_idx_iso ON estados (idpais,iso);
INSERT INTO estados (nombre,iso,fips,admin,refs,idpais)
                    (SELECT name,iso,fips,admin,reference,idpais
                       FROM tmp_estados());
DROP FUNCTION tmp_estados();
-- Ajustar la secuentcia.
-- Tal vez sea buena idea "saber" que los nuevos estados agregados (si los hay) serán del 5,000 en adelante
ALTER SEQUENCE estados_idestado_seq RESTART 5000;
-- o si no
-- SELECT setval('estados_idestado_seq',(SELECT max(idestado) FROM estados),TRUE);

6.Ajustes para México

```sql
UPDATE estados SET nombre = 'Ciudad de México', iso = 'CMX' WHERE (idpais,iso)=(484,'DIF');
UPDATE estados SET nombre = 'Guerrero' WHERE (idpais,iso)=(484,'GRO');
UPDATE estados SET nombre = 'Hidalgo' WHERE (idpais,iso)=(484,'HID');
UPDATE estados SET nombre = 'México' WHERE (idpais,iso)=(484,'MEX');
UPDATE estados SET nombre = 'Michoacán de Ocampo' WHERE (idpais,iso)=(484,'MIC');
```

7. Prueba

```sql
SELECT p.idpais, p.nombre AS pais, p.iso AS pais_iso,
       e.idestado, e.nombre AS estado, e.iso AS estados_iso
  FROM paises AS p 
  LEFT JOIN estados AS e USING (idpais)
 WHERE p.iso='MX' ORDER BY estado;
```
