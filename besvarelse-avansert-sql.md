# Besvarelse: Avansert SQL

## Oppgave 1: Window Functions

### Del 1: Forklar SQL-spørringene

1.  **Spørring:**
    ```sql
    SELECT
        Fornavn,
        Etternavn,
        Årslønn,
        RANK() OVER (ORDER BY Årslønn DESC) AS Lønnsrangering
    FROM Ansatt;
    ```
    **Forklaring:**
    *   Henter ut fornavn, etternavn, årslønn fra Ansatt tabellen og rangerer de etter synkende årslønn

2.  **Spørring:**
    ```sql
    SELECT
        V.Betegnelse,
        K.Navn AS Kategori,
        V.Pris,
        AVG(V.Pris) OVER (PARTITION BY K.Navn) AS GjennomsnittsprisForKategori
    FROM Vare V
    JOIN Kategori K ON V.KatNr = K.KatNr;
    ```
    **Forklaring:**
    *   Henter ut varer kategorisert alfabetisk ut i fra hvilken kategori de hører til og henter ut gjennnomsnittspris for den kategorien.

### Del 2: Lag SQL-spørringer

1.  **Rangering av varer per kategori:**
    ```sql
    SELECT
        V.Betegnelse,
        K.Navn,
        V.Antall,
        RANK() OVER (PARTITION BY K.Navn ORDER BY V.Antall DESC) AS Rangering
    FROM Vare V
    JOIN Kategori K ON V.KatNr = K.KatNr
    ```

2.  **Løpende sum av ordrebeløp:**
    ```sql
    SELECT
        O.OrdreNr,
        O.OrdreDato,
        SUM(OL.PrisPrEnhet * OL.Antall) AS Ordrebeløp,
        SUM(SUM(OL.PrisPrEnhet * OL.Antall))
            OVER (ORDER BY O.OrdreDato) AS LopendeSum
    FROM Ordre O
    JOIN Ordrelinje OL ON O.OrdreNr = OL.OrdreNr
    GROUP BY O.OrdreNr, O.OrdreDato
    ORDER BY O.OrdreDato;
    ```

3.  **Prosentandel av kategoriprisen:**
    ```sql
    SELECT
        V.Betegnelse,
        K.Navn AS Kategori,
        V.Pris,
        ROUND((V.Pris / SUM(V.Pris) OVER (PARTITION BY K.Navn)) * 100, 2) AS Prosentandel
    FROM Vare V
    JOIN Kategori K ON V.KatNr = K.KatNr;
    ```

---

## Oppgave 2: Common Table Expressions (CTEs)

### Del 1: Forklar SQL-spørringen

1.  **Spørring:**
    ```sql
    WITH KunderPerPoststed AS (
        SELECT PostNr, COUNT(*) AS AntallKunder
        FROM Kunde
        GROUP BY PostNr
    )
    SELECT P.Poststed, KPP.AntallKunder
    FROM Poststed P
    JOIN KunderPerPoststed KPP ON P.PostNr = KPP.PostNr
    WHERE KPP.AntallKunder > 5
    ORDER BY KPP.AntallKunder DESC;
    ```
    **Forklaring:**
    *   Henter ut poststeder med antall registrerte kunder over 5 

### Del 2: Lag SQL-spørringer

1.  **Ansatte med over gjennomsnittslønn:**
    ```sql
    SELECT
        Fornavn,
        EtterNavn,
        Årslønn
    FROM Ansatt
    WHERE Årslønn > (
        SELECT AVG(Årslønn)
        FROM Ansatt
    );
    ```

2.  **Kategorier med flest varer:**
    ```sql
    SELECT
        K.Navn,
        COUNT(*) AS AntallVarer
    FROM Vare V
    JOIN Kategori K ON V.KatNr = K.KatNr
    GROUP BY K.Navn
    ORDER BY AntallVarer DESC;
    ```

3.  **Rekursiv CTE - Hierarki av ansatte:**
    ```sql
    ALTER TABLE Ansatt
    ADD LederAnsNr INTEGER;
    
    ALTER TABLE Ansatt
    ADD CONSTRAINT AnsattLederFN
    FOREIGN KEY (LederAnsNr) REFERENCES Ansatt(AnsNr);
    
    WITH RECURSIVE hierarki AS (
    SELECT AnsNr, Fornavn, LederAnsNr, 0 AS Nivå
    FROM Ansatt
    WHERE LederAnsNr IS NULL

    UNION ALL

    SELECT A.AnsNr, A.Fornavn, A.LederAnsNr, H.Nivå + 1
    FROM Ansatt A
    JOIN hierarki H ON A.LederAnsNr = H.AnsNr
    )
    SELECT * FROM hierarki;
    ```

---

## Oppgave 3: Avanserte Subqueries

### Del 1: Forklar SQL-spørringene

1.  **Spørring (Correlated Subquery):**
    ```sql
    SELECT V.Betegnelse, V.Pris
    FROM Vare V
    WHERE V.Pris > (
        SELECT AVG(Pris)
        FROM Vare
        WHERE KatNr = V.KatNr
    );
    ```
    **Forklaring:**
    *   Den henter ut alle varer som har en høyere pris enn gjennomsnittsprisen i den kategorien

2.  **Spørring (Subquery i `FROM`):**
    ```sql
    SELECT Kategori, Gjennomsnittspris
    FROM (
        SELECT K.Navn AS Kategori, AVG(V.Pris) AS Gjennomsnittspris
        FROM Vare V
        JOIN Kategori K ON V.KatNr = K.KatNr
        GROUP BY K.Navn
    ) AS KategoriPriser
    WHERE Gjennomsnittspris > 100;
    ```
    **Forklaring:**
    *   Henter ut alle kategorier og gjennomsnittsprisen på varene i hver kategori

### Del 2: Lag SQL-spørringer

1.  **Kunder som har bestilt en spesifikk vare:**
    ```sql
    SELECT K.Fornavn, K.Etternavn, V.Betegnelse
    FROM Kunde K
    JOIN Ordre O ON K.KNr = O.KNr
    JOIN Ordrelinje OL ON O.OrdreNr = OL.OrdreNr
    JOIN Vare V ON OL.VNr = V.VNr
    WHERE K.KNr IN (
        SELECT O.KNr
        FROM Ordre O
        JOIN Ordrelinje OL ON O.OrdreNr = OL.OrdreNr
        WHERE OL.VNr = '10820'
    )
    AND V.VNr = '10820';
    ```

2.  **`EXISTS` - Kategorier med dyre varer:**
    ```sql
    SELECT K.Navn
    FROM Kategori K
    WHERE EXISTS (
        SELECT 1
        FROM Vare V
        WHERE V.KatNr = k.KatNr
            AND v.Pris > 1000
    );
    ```

3.  **Varer dyrere enn gjennomsnittet:**
    ```sql
    SELECT Betegnelse, Pris
    FROM Vare
    WHERE Pris > (
    SELECT AVG(Pris)
    FROM Vare
    )
    ORDER BY Pris DESC;
    ```
