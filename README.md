# test-cle-sql
### Tests d'utilisation de clés artificielles vs clés naturelles composées

Il y a un débat qui sévit depuis longtemps entre les personnes qui utilisent des clés naturelles composées et les personnes qui utilisent des clé artificielles non-composé (autoincrement). L'utilisation des clés artificielles semblent être de plus en plus la "norme" dans l'industrie. J'ai cependant entendu à plusieurs reprises des personnes alléger que les clées naturelles composées était plus performante que les clés artificielles. Voici donc une expérience que j'ai réalisé avec PostgreSQL.

Script de nettoyage (facultatif)
``` sql
DROP TABLE article_naturel;
DROP TABLE facture_naturel;
DROP TABLE client_naturel;

DROP TABLE article_artificiel;
DROP TABLE facture_artificiel;
DROP TABLE client_artificiel;
```

Création des tables de tests
``` sql
-------------------------------------
-- TABLE AVEC CLÉS NATURELLES
-------------------------------------
CREATE TABLE client_naturel
(
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    CONSTRAINT client_naturel_pk PRIMARY KEY (nom, prenom, telephone, codepostal)
);

CREATE TABLE facture_naturel
(
    _date VARCHAR(10),
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    CONSTRAINT facture_naturel_pk PRIMARY KEY (_date, nom, prenom, telephone, codepostal),
	FOREIGN KEY (nom, prenom, telephone, codepostal) REFERENCES client_naturel(nom, prenom, telephone, codepostal)
);

CREATE TABLE article_naturel
(
    nom_article VARCHAR(10),
    _date VARCHAR(10),
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    CONSTRAINT article_naturel_pk PRIMARY KEY (nom_article, _date, nom, prenom, telephone, codepostal),
	FOREIGN KEY (_date, nom, prenom, telephone, codepostal) REFERENCES facture_naturel(_date, nom, prenom, telephone, codepostal)
);

-------------------------------------
-- TABLE AVEC CLÉ ARTIFICIELLE
-------------------------------------
CREATE TABLE client_artificiel
(
    client_id SERIAL,
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    CONSTRAINT client_artificiel_pk PRIMARY KEY (client_id)
);

CREATE TABLE facture_artificiel
(
    facture_id SERIAL,
    _date VARCHAR(10),
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    client_id INT,
    CONSTRAINT facture_artificiel_pk PRIMARY KEY (facture_id),
	FOREIGN KEY (client_id) REFERENCES client_artificiel(client_id)
);

CREATE TABLE article_artificiel
(
    article_id SERIAL,
    nom_article VARCHAR(10),
    _date VARCHAR(10),
    nom VARCHAR(10),
    prenom VARCHAR(10),
    telephone VARCHAR(10),
    codepostal VARCHAR(10),
    facture_id INT,
    CONSTRAINT article_artificiel_pk PRIMARY KEY (nom_article),
	FOREIGN KEY (facture_id) REFERENCES facture_artificiel(facture_id)
);
```

On génère des données aléatoires pour chacunes des tables :
- Table client (100 lignes)
- Table facture (10 000 lignes)
- Table article (100 000 lignes)

Les données sont exactement les mêmes pour les tables de clés naturelles et artificielles

``` sql
-- Fonction utilitaire qui génère des chaînes aléatoires de 10 caractères alphas majuscules
CREATE OR REPLACE FUNCTION generate_string()
    RETURNS VARCHAR(10) AS $$
    DECLARE
        chars VARCHAR = '';
        i INT = 0;
    BEGIN
        FOR i IN 0..9 LOOP
            chars = chars || CHR(64 + (trunc((random() * 25 + 1))) :: INT);
        END LOOP;

        RETURN chars;
    END;
$$ LANGUAGE plpgsql;


-- Crée les enregistrements dans la BD avec des valeurs aléatoires.
CREATE OR REPLACE FUNCTION populate_datatabase()
    RETURNS void AS $$
    DECLARE
        i integer := 0;
        j integer := 0;
        k integer := 0;

        nom VARCHAR(10);
        prenom VARCHAR(10);
        telephone VARCHAR(10);
        codepostal VARCHAR(10);
        _date VARCHAR(10);
        nom_article VARCHAR(10);
    BEGIN
        FOR i IN 1..100 LOOP
            nom = generate_string();
            prenom = generate_string();
            telephone = generate_string();
            codepostal = generate_string();

            INSERT INTO client_naturel (nom, prenom, telephone, codepostal) VALUES (nom, prenom, telephone, codepostal);
            INSERT INTO client_artificiel (nom, prenom, telephone, codepostal) VALUES (nom, prenom, telephone, codepostal);

            FOR j IN 1..100 LOOP
                _date = generate_string();
                INSERT INTO facture_naturel (_date, nom, prenom, telephone, codepostal) VALUES (_date, nom, prenom, telephone, codepostal);
                INSERT INTO facture_artificiel (_date, nom, prenom, telephone, codepostal, client_id) VALUES (_date, nom, prenom, telephone, codepostal, (select max(client_id) from client_artificiel));

                FOR k IN 1..10 LOOP
                    nom_article = generate_string();
                    INSERT INTO article_naturel (nom_article, _date, nom, prenom, telephone, codepostal) VALUES (nom_article, _date, nom, prenom, telephone, codepostal);
                    INSERT INTO article_artificiel (nom_article, _date, nom, prenom, telephone, codepostal, facture_id) VALUES (nom_article, _date, nom, prenom, telephone, codepostal, (select max(facture_id) from facture_artificiel));
                END LOOP;
            END LOOP;
        END LOOP;
    END;
    $$ LANGUAGE plpgsql;

select populate_datatabase();
COMMIT;
```

Test avec jointures sur les 3 tables (client -> facture -> article)

``` sql
-- Clé naturelle
EXPLAIN ANALYZE
    SELECT *
    FROM client_naturel c
    JOIN facture_naturel f
        ON c.nom = f.nom AND c.prenom = f.prenom AND c.telephone = f.telephone AND c.codepostal = f.codepostal
    JOIN article_naturel a
        ON f._date = a._date AND f.nom = a.nom AND f.prenom = a.prenom AND f.telephone = a.telephone AND f.codepostal = a.codepostal;
-- 1er  test : Execution time: 71015.921 ms
-- 2eme test : Execution time: 70957.549 ms
-- 3eme test : Execution time: 70832.049 ms

-- Clé artificielle
EXPLAIN ANALYZE
    SELECT *
    FROM client_artificiel c
    JOIN facture_artificiel f
        ON c.client_id = f.client_id
    JOIN article_artificiel a
        ON f.facture_id = a.facture_id;
-- 1er  test : Execution time: 39.235 ms
-- 2eme test : Execution time: 38.744 ms
-- 3eme test : Execution time: 39.260 ms
```
On voit que le temps d'exécution pour les clés naturelles est extrêmements plus élevé que pour les clés artificielles. Le problème vient surement du fait qu'il n'y a pas d'index pour chacune des colonnes de clé composé. Remédions à ce problème :

``` sql
CREATE INDEX ON client_naturel (nom);
CREATE INDEX ON client_naturel (prenom);
CREATE INDEX ON client_naturel (telephone);
CREATE INDEX ON client_naturel (codepostal);

CREATE INDEX ON facture_naturel (_date) ;
CREATE INDEX ON facture_naturel (nom) ;
CREATE INDEX ON facture_naturel (prenom) ;
CREATE INDEX ON facture_naturel (telephone) ;
CREATE INDEX ON facture_naturel (codepostal) ;

CREATE INDEX ON article_naturel (_date);
CREATE INDEX ON article_naturel (nom);
CREATE INDEX ON article_naturel (prenom);
CREATE INDEX ON article_naturel (telephone);
CREATE INDEX ON article_naturel (codepostal);
```

Nouveau test avec les indexes :
``` sql
EXPLAIN ANALYZE
    SELECT *
    FROM client_naturel c
    JOIN facture_naturel f
        ON c.nom = f.nom AND c.prenom = f.prenom AND c.telephone = f.telephone AND c.codepostal = f.codepostal
    JOIN article_naturel a
        ON f._date = a._date AND f.nom = a.nom AND f.prenom = a.prenom AND f.telephone = a.telephone AND f.codepostal = a.codepostal;
-- 1er  test : Execution time: 116.523 ms
-- 2eme test : Execution time: 114.294 ms
-- 3eme test : Execution time: 114.704 ms
```

Le temps d'exécution est nettement meilleur, mais quand même 3 fois plus lent qu'avec les clés artificielles. De plus, il a fallut créer 14 indexes supplémentaires pour gérer seulement 3 tables.

On pourrait cependant dire que l'utilisation de clés composés permettrait de chercher tous les articles qu'un client a achetés sans faire aucune jointure sur les factures.

``` sql
EXPLAIN ANALYZE
    SELECT *
        FROM client_naturel c
        JOIN article_naturel a
            ON c.nom = a.nom AND c.prenom = a.prenom AND c.telephone = a.telephone AND c.codepostal = a.codepostal;
-- Execution time: 59.039 ms
-- Execution time: 50.664 ms
-- Execution time: 51.245 ms
```

Le temps d'exécution demeure meilleure en utilisant des clés artificielles en joingnant 3 tables, plutôt que 2 tables avec des clés composé.

En plus de la rapidité, les clés artificielles comportent les avantages suivants :
- Permet de gérer les changements fonctionnels à l'intérieur d'une organisation (changement de direction, changement de législation, etc.)
- Permet d'écrire des requêtes SQL beaucoup plus courtes
- Plus facile de reconnaître la clé primaire d'une table (première colonne, se termine par '\_id'), plus besoin de consulter le schéma à chaque fois.

À la lumière de ces tests, je ne suis pas certain de comprendre pourquoi certaines personnes déconseillent l'utilisation de clés artificielle pour la plupart des tables. Peut-être ai-je fait une erreur dans mon analyse ?
