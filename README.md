  # BusinessConsultingProjekte
Dies ist ein Repository, welches ausgewählte Projekte während meiner Anstellung als Business Consultant bei der Capita Customer Services AG enthält.

# Mein Portfoliooo

Willkommen zu meinem Portfolio! Hier finden Sie Informationen zu meinen Projekten und beruflichen Erfahrungen.

## Projekt: Berechnung des Bradford-Faktors zur Mitarbeiterabwesenheit

### Beschreibung
Dieses Projekt beinhaltet die Berechnung des Bradford-Faktors für Mitarbeiter basierend auf ihren Abwesenheitsdaten. Der Bradford-Faktor ist eine Kennzahl, die die Anzahl und Dauer der Krankheitsausfälle von Mitarbeitern bewertet. Diese Metrik hilft dabei, die Auswirkungen von häufigen, kurzen Abwesenheiten zu analysieren, die sich stärker auf den Betrieb auswirken können als längere, seltenere Ausfälle.

### SQL-Code
Der folgende SQL-Code wird verwendet, um die relevanten Daten zu extrahieren, zu verarbeiten und die Bradford-Faktoren zu berechnen:

```sql
--Deklaration der Zeitspanne
DECLARE @Von DATE = '2024-02-01'
DECLARE @Bis DATE = '2024-02-29'

DECLARE @i INT
DECLARE @j DATE

--Erstellen temporärer Tabellen
CREATE TABLE #Result_Periode (st_staff INT, Datum DATETIME, Zähler INT, Krank INT)
CREATE TABLE #Schichtcenter_Daten (Eintraege_total INT, Eintraege_MA INT, Datum DATETIME, st_staff_id INT, Krank INT)
CREATE TABLE #Bradford_Ergebnisse (st_staff_id INT, Anzahl_Ausfaelle INT, Anzahl_Fehltage INT)

--Einfügen von Daten in #Schichtcenter_Daten
INSERT INTO #Schichtcenter_Daten
SELECT ROW_NUMBER() OVER (ORDER BY tws.st_staff_id) AS Eintraege_total,
       ROW_NUMBER() OVER (PARTITION BY tws.st_staff_id ORDER BY tws.on_date, tws.st_staff_id) AS Eintraege_MA,
       tws.on_date,
       tws.st_staff_id,
       SUM(CASE WHEN tkt.name NOT LIKE '%Schichtfrei%' THEN 1 ELSE 0 END) OVER (PARTITION BY tws.st_staff_id) AS Anzahl_Krank
FROM [c1db3001].[isps_iewfm].[isps].tw_schedule tws
JOIN [c1db3001].[isps_iewfm].[isps].tk_type tkt ON tkt.tk_type_id = tws.ref_id
WHERE tws.on_date BETWEEN @Von AND @Bis 
AND tws.layer = -1 
AND tws.version_id = 1000 
AND tws.level_id = 3000
AND tkt.class = 2 AND tkt.name NOT LIKE '%Unfall%' AND tkt.is_deleted = 0
GROUP BY tws.on_date, tws.st_staff_id, tkt.name
ORDER BY tws.on_date, tws.st_staff_id

--Verarbeiten der Daten
SET @i = 1

WHILE @i <= (SELECT MAX(Eintraege_total) FROM #Schichtcenter_Daten)
BEGIN
       IF ((SELECT Eintraege_MA FROM #Schichtcenter_Daten WHERE Eintraege_total = @i) = 1 
           OR (SELECT Datum FROM #Schichtcenter_Daten WHERE Eintraege_total = @i) <> DATEADD(DAY,1,@j))
       BEGIN
             INSERT INTO #Result_Periode
                    SELECT st_staff_id, Datum, 1, Krank 
                    FROM #Schichtcenter_Daten 
                    WHERE Eintraege_total = @i
       END
       SET @j = (SELECT Datum FROM #Schichtcenter_Daten WHERE Eintraege_total = @i)
       SET @i = @i + 1
END

--Einfügen der Bradford-Ergebnisse
INSERT INTO #Bradford_Ergebnisse
SELECT RP.st_staff, SUM(RP.zähler) AS Anzahl_Ausfaelle, AVG(RP.Krank) AS Anzahl_Fehltage 
FROM #Result_Periode RP 
GROUP BY RP.st_staff

--Ausgabe der Ergebnisse
SELECT * FROM #Bradford_Ergebnisse

--Bereinigung der temporären Tabellen
DROP TABLE #Bradford_Ergebnisse, #Result_Periode, #Schichtcenter_Daten

