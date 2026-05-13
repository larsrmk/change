
---
---
---

---
---
---

# 1.0 Skalierung von GitOps
## 1.1 Ausgangslage
Im folgenden Szenario betreiben wir 10 Kubernetes-Cluster mit jeweils vier Stages:
- main
- test
- integra 
- prod

Wenn pro Cluster noch zwischen einer internen und einer externen Umgebung unterschieden werden soll, wächst die Infrastruktur nochmals.

##### Wir evaluieren die folgenden vier Modelle:
- Modell 1: Branch per Stage
- Modell 2: Repo per Stage
- Modell 3: Folder per Stage
- Modell 4 OCI-Registry

##### Technologie-Stack
- Gitea
- Kargo
- ArgoCD
- Habour

---
---
## 1.2 Modell 1: Branch per Stage
#### Struktur
Für jedes Kubernetes-Cluster wird ein eigenes Repository in Gitea betrieben. Die vier Stages werden in den Repos als Branches abgebildet.
```plain
Gitea Repository: cluster-01
 ├── Branch: main (Base)
 ├── Branch: test
 ├── Branch: integra
 └── Branch: prod
```
#### Rollenverteilung der Tools
##### Gitea: 
Änderungen von Entwicklern werden auf dem main-Branch durchgeführt.
Die Branches test, integra und prod werden nur von Kargo verwaltet.
##### Kargo: 
Das Warehouse in einem Kargo-project erkennt die Änderungen auf dem main-Branch als Freights und pusht diese dann auf die verschiedenen Branches der verschiedenen Stages.
##### Argo CD: 
Überwacht die einzelnen Branches.
#### Bewertung
- Umgebungsspezifische Werte (z. B. Datenbankpfade) können versehentlich schnell überschrieben werden
- Komplexe Änderungshistorie

---
---
## 1.3 Modell 2: Repo per Stage
#### Struktur
Für jede Stage wird ein eigenes physisch getrenntes Repository angelegt in Gitea angelegt.
```plain
Gitea Server/
 ├── Repo: cluster-01-base
 ├── Repo: cluster-01-test
 ├── Repo: cluster-01-integra
 └── Repo: cluster-01-prod
```
#### Rollenverteilung der Tools
##### Gitea: 
Änderungen eines Entwicklers werden direkt auf den main-Branch des base-Repos gepusht.
##### Kargo
Das Warehouse in Kargo schaut auf den main-Branch und gibt die Änderungen nicht mehr an den nächsten Branch weiter sondern in das nächste Repository.
##### ArgoCD
Die verschiedenen Stages/Umgebungen werden durch separate Repositories erstellt.
#### Bewertung
- Sehr hohe Sicherheit: Berechtigungen und Zugänge können für jedes Repo einzeln gesetzt werden
- Allerdings sehr viele Repos die verwaltet werden müssen

---
---
## 1.4 Modell 3: Folder per Stage

#### Struktur
Es gibt für jedes Kubernetes-Cluster ein eigenes Repository in Gitea. Die verschiedenen Stages existieren als Ordner auf dem main-Branch.
```plain
Gitea Repository: enterprise-gitops (Branch: main)
 └── clusters/
     ├── cluster-01/
     │   ├── base/      # Einstiegspunkt für Entwickler
     │   ├── test/      # Zielordner (Kargo schreibt, Argo CD liest)
     │   ├── integra/   # Zielordner (Kargo schreibt, Argo CD liest)
     │   └── prod/      # Zielordner (Kargo schreibt, Argo CD liest)
```
#### Rollenverteilung der Tools
##### Gitea
Änderungen werden im base-Ordner durchgeführt und dann auf den main-Branch gepusht. Die Ordner für die Stages werden durch das Branch-Protection-Feature und das CODEOWNERS-Feature geschützt.
##### Kargo
Das Warehouse schaut auf den base-Ordner und legt Änderungen als Freights an. Die Änderungen überschreiben die Ordner für die einzelnen stages.
##### Argo CD
Schaut auf die einzelnen Ordner und synchronisiert die Cluster mit den jeweiligen Unterordnern in Gitea.

#### Bewertung
- Gesamter Code eines Clusters ist auf einen Blick sichtbar
- Gitea = Single Point of Failure
- Kann unübersichtlich werden
- Zugriffskontrolle über Branch-Protection-Feature und CODEOWNERS-Datei komplizierter umzusetzen -> Fehleranfälliger

---
---
## 1.5 Modell 4: OCI-Registry
#### Struktur
Es gibt für jedes Kubernetes-Cluster nur noch ein Repo mit einem main-Branch. Fertige Konfigurationen wandern als Artifakte in eine OCI-Registry.
```plain
Harbor Registry (Das Zentrallager)
 ├── Projekt: cluster-01/
 │   ├── Paket: base-config    (Tag: 1.0.0)
 │   ├── Paket: test-config    (Tag: 1.0.0)
 │   ├── Paket: integra-config (Tag: 0.9.5)
 │   └── Paket: prod-config    (Tag: 0.9.0)
```
#### Rollenverteilung der Tools
##### Gitea
Änderungen von Entwicklern werden auf den main-Branch gepusht.
##### OCI-Registry
Eine CI-Pipeline (bspw. Tekton) verpackt den Code aus Gitea zu fertigen Paketen und legt sie in der Registry ab. Dort werden diese gescannt und signiert.
##### Kargo
Die Pakete aus der Registry werden von Kargo mit einem label versehen und an die nächste Stage weitergegen. Kargo hat mit Gitea nichts mehr zu tun.
##### ArgoCD
Holt sich die fertigen signierten Pakete aus der Registry und wendet sie auf den verschiedenen Umgebungen an

#### Bewertung
- Cluster selber haben keinen Zugriff auf Quellcode -> hohe Sicherheit
- Signierte Pakete in der OCI-Registry schließen Manipulation in prod so gut wie aus
- OCI-Registry = Single point of Failure 
- Keinen Zugriff auf den Quellcode in Prod

---
---
---
# 2.0 Workflow
## 2.1 Architektur und Rollenverteilung (Der Tech-Stack)
Dieser Workflow beschreibt eine hochskalierte „Gitless GitOps“-Architektur (OCI-basiert) für 10 Kubernetes-Cluster mit jeweils 4 Stages (Base, Test, Integra, Prod). 

Alle Tools sind Kubernetes-nativ, Open Source und werden über Helm-Charts auf einem zentralen Management-Cluster betrieben.

*   **Gitea (Das Konstruktionsbüro):** Das Quellcode-Repository. Der einzige Ort, an dem Menschen arbeiten.
*   **Tekton (Die Fabrik):** Die CI-Pipeline. Ein Kubernetes-natives Tool, das als Pods läuft, aus Quellcode fertige Pakete (OCI-Artefakte) baut und sich danach selbst beendet.
*   **Harbor (Der Supermarkt / Der Tresor):** Das OCI-Zentrallager. Speichert die Pakete sicher und signiert ab.
*   **Kargo (Der Logistiker):** Arbeitet unsichtbar im Hintergrund und schiebt Freigaben (Tags) durch die Stages innerhalb von Harbor.
*   **Argo CD (Der Filialleiter):** Sitzt auf den 10 Ziel-Clustern, zieht die Pakete aus Harbor und wendet sie an.

---
---
## 2.2 Der Ablauf (Schritt für Schritt)

### 2.2.1 Schritt 1: Der Entwickler arbeitet 
*Szenario: Ein Entwickler möchte der Applikation `backend-api` mehr Arbeitsspeicher zuweisen.*
1. Der Entwickler öffnet **Gitea**.
2. Er erstellt einen neuen Feature-Branch (z. B. `feature/mehr-ram`).
3. Er ändert die Datei `environments/base/backend-api-values.yaml` und trägt dort `memory: 2Gi` ein.
4. Er erstellt einen **Pull Request (PR)** gegen den `main`-Branch.
5. Ein Kollege (Codeowner) prüft den PR, gibt ihn frei (Approve) und mergt ihn in den `main`-Branch.

*(Ab hier ist die Arbeit des Entwicklers beendet. Der Rest läuft automatisiert, unterbrochen nur durch manuelle Quality-Gate-Freigaben für Kargo.)*

---
### 2.2.2 Schritt 2: Tekton baut und verpackt (Automatisiert)
Der Merge in den `main`-Branch von Gitea löst einen Webhook aus. Dieser weckt **Tekton** auf. Tekton startet dynamisch kleine Pods auf dem Cluster, die folgende `Tasks` abarbeiten:
1. **Task "Clone":** Tekton klont das Gitea-Repository.
2. **Task "Render":** Tekton führt `helm template` (oder Kustomize) aus, nimmt das Basis-Chart der App und mischt es mit den Werten aus der `base`-Datei zu einem fertigen, lesbaren Kubernetes-Manifest.
3. **Task "Package & Push":** Tekton verpackt das fertige Manifest in ein OCI-Archiv und pusht es in die **Harbor Registry** in das Projekt `cluster-01` unter dem Namen `backend-api` mit dem Initial-Tag `1.0.5-test`.

---
### 2.2.3 Schritt 3: Kargo erkennt das neue Paket (Automatisiert)
**Kargo** hat in seinem "Warehouse" konfiguriert, dass es das Harbor-Projekt `cluster-01` nach neuen Paketen überwacht.
1. Kargo bemerkt das neu aufgetauchte Paket `1.0.5-test` in Harbor.
2. Kargo erstellt einen "Freight" daraus.
3. Da die `test`-Stage oft auf "auto-promotion" gestellt ist, gibt Kargo dieses Freight sofort für die Test-Umgebung in Harbor frei.

---
### 2.2.4 Schritt 4: Argo CD rollt in die Testumgebung aus (Automatisiert)
Auf dem ersten Zielcluster (Test) läuft **Argo CD**.
1. Argo CD ist so konfiguriert, dass es dauerhaft den Tag `1.0.5-test` im Harbor-Projekt `cluster-01` überwacht.
2. Argo CD lädt das von Kargo freigegebene Paket herunter.
3. Argo CD wendet die Konfiguration (`memory: 2Gi`) sofort auf dem Test-Cluster an.
4. *(Optional: Tekton könnte nun automatisierte End-to-End-Tests auf dem Test-Cluster anstoßen).*

---
### 2.2.5 Schritt 5: Promotion nach Integra 
Die Test-Umgebung läuft stabil. Das QA-Team (Quality Assurance) soll die Änderung prüfen.
1. Ein QA-Mitarbeiter öffnet die grafische Oberfläche von **Kargo**.
2. Er sieht visuell, dass das Paket `1.0.5` in der `test`-Stage erfolgreich war, aber für `integra` noch bereitliegt.
3. Er klickt in Kargo manuell auf den Button **"Promote to Integra"**.

---
### 2.2.6 Schritt 6: Kargo und Argo CD arbeiten in Integra (Automatisiert)
1. **Kargo** manipuliert das OCI-Paket in Harbor: Es nimmt das bestehende Paket und versieht es zusätzlich mit dem Tag `1.0.5-integra`. *(Es wird kein Code mehr neu gebaut, das exakt selbe Artefakt wird wiederverwendet – dies garantiert "Immutability").*
2. Das **Argo CD** auf dem Integra-Cluster überwacht den Integra-Tag in Harbor. Es bemerkt das Update, lädt das Paket herunter und wendet es an.

---
### 2.2.7 Schritt 7: Promotion nach Prod
Die QA hat das Integra-System erfolgreich getestet. Das Release-Team gibt grünes Licht.
1. Der Release-Manager öffnet die **Kargo**-UI.
2. Er klickt auf **"Promote to Prod"**.

---
### 2.2.8 Schritt 8: Der finale Rollout in Prod (Automatisiert)
1. **Kargo** versieht das Harbor-Paket mit dem finalen Tag `1.0.5-prod`.
2. Das **Argo CD** im Produktions-Cluster (welches sicherheitstechnisch in einer isolierten DMZ steht) hat keinen Zugriff auf Gitea. Es überwacht nur Harbor.
3. Es bemerkt das neue Prod-Paket in Harbor, zieht es und gibt dem Backend-Server live die 2Gi RAM.

---
---
## 2.3 Zusammenfassung der Übergaben

Der Wasserfall-Ablauf der Verantwortlichkeiten:
1. **Entwickler** $\rightarrow$ schreibt Code in $\rightarrow$ **Gitea**
2. **Gitea** $\rightarrow$ triggert (via Webhook) $\rightarrow$ **Tekton**
3. **Tekton** $\rightarrow$ baut Manifeste und pusht OCI-Paket nach $\rightarrow$ **Harbor**
4. **Harbor** $\rightarrow$ wird überwacht von $\rightarrow$ **Kargo**
5. **Kargo** $\rightarrow$ schiebt Tags zwischen Stages innerhalb von $\rightarrow$ **Harbor**
6. **Harbor** $\rightarrow$ wird gelesen von $\rightarrow$ **Argo CD**
7. **Argo CD** $\rightarrow$ wendet an auf $\rightarrow$ **Kubernetes Ziel-Cluster**

**Fazit:** 
Durch diese Architektur erreichen wir eine perfekte "Separation of Concerns" (Gewaltenteilung). Es gibt keinen maschinellen Git-Churn in Gitea, kein Entwickler kann die Produktion versehentlich lahmlegen, und externe Cluster sind netzwerktechnisch vollständig vom internen Quellcode isoliert.

---
---
---
# 3.0 Hub-and-Spoke GitOps Architektur


Diese Dokumentation beschreibt das Architektur-Design und den Workflow für eine Kubernetes-Landschaft bestehend aus einem zentralen Management-Cluster (Hub) und mehreren isolierten Workload-Clustern (Spokes). 

Der Kern dieses Designs ist das **GitOps Pull-Modell**: Die Ziel-Cluster sind autarke "Konsumenten", die keinerlei Build- oder Logistik-Tools hosten.

## 3.1 Architektur-Komponenten: Wer macht was?

### 3.1.1 Der Hub (Management-Cluster / "Die Fabrik")
Hier läuft die gesamte Toolchain für Code-Verwaltung, Build-Prozesse und globale Logistik.
* **Gitea:** Quellcode-Repository (Die "Single Source of Truth" für den Code).
* **Tekton:** Führt CI-Pipelines aus (baut Images und verpackt Kubernetes-Manifeste zu OCI-Paketen).
* **Master-Harbor:** Die zentrale OCI-Registry. Nimmt alle Builds von Tekton an.
* **Kargo:** Das CD-Logistik-Tool. Steuert die Promotions (z.B. Test $\rightarrow$ Integra $\rightarrow$ Prod) durch reines Tagging innerhalb des Master-Harbors.
* **Argo CD (Global):** Optional, um die Infrastruktur der Cluster selbst zu verwalten.
* **Cert-Manager:** Kümmert sich um SSL/TLS-Zertifikate.

---
### 3.1.2 Die Spokes (Ziel-Cluster wie Test, Integra, Prod / "Die Filialen")
Die Ziel-Cluster sind maximal verschlankt. **Hier laufen absichtlich KEIN Gitea, KEIN Tekton und KEIN Kargo.**
Auf jedem Spoke läuft ausschließlich:
* **Lokales Argo CD:** Der lokale Agent. Überwacht nur den lokalen Harbor auf dem gleichen Cluster und zieht (pullt) fertige Pakete.
* **Lokaler Harbor (als Proxy Cache):** Ein leichtgewichtiges Harbor-Setup ohne eigene Nutzer. Er verweist auf den Master-Harbor als "Endpoint" und puffert angefragte Images und OCI-Pakete lokal.

---

## 3.2 Der Workflow: Von Gitea in die Produktion

*Szenario: Wir rollen die Applikation `backend-api` (Version 1.0.5) in die Stage `Prod` aus.*

### 3.2.1 Phase 1: Build & Promote (Komplett auf dem Hub)
1. **Commit:** Ein Entwickler pusht eine Änderung nach **Gitea**.
2. **Build:** **Tekton** wird getriggert, baut das Container-Image sowie das OCI-Manifest-Paket und pusht beides in den **Master-Harbor**. *(Tekton beendet sich danach wieder).*
3. **Promotion:** Ein Release-Manager klickt in **Kargo** auf "Promote to Prod". Kargo markiert das fertige Paket im Master-Harbor mit dem Tag `1.0.5-prod`.

*(Bis hierhin ist auf den Ziel-Clustern absolut nichts passiert. Die gesamte Last lag auf dem Hub.)*

---
### 3.2.2 Phase 2: Der Pull & Cache (Komplett auf dem Spoke)
1. **Sync-Versuch:** Das **lokale Argo CD** auf dem Produktions-Cluster sucht in seinem *lokalen Harbor* nach dem Tag `1.0.5-prod`.
2. **Download (Proxying):** Der lokale Harbor stellt fest, dass er das Paket noch nicht hat. Er verbindet sich mit dem Master-Harbor (Hub), lädt das Paket `1.0.5-prod` herunter und **speichert es in seinem lokalen Cache ab**.
3. **Auslieferung & Deployment:** Der lokale Harbor übergibt das Paket an das lokale Argo CD, welches die Konfiguration sofort auf dem Cluster anwendet.
4. **Image Pull:** Die Kubernetes-Worker-Nodes des Spoke-Clusters starten die neuen Pods. Sie fragen den *lokalen Harbor* nach dem Docker-Image. Auch dieses wird einmalig vom Master geladen, lokal gepuffert und an die Nodes ausgeliefert.

---
### 3.2.3 Phase 3: Zukünftige Anfragen (Die Autarkie)
Wenn ein Kubernetes-Node auf dem Spoke-Cluster neu gestartet wird und das Image erneut ziehen muss, schaut der lokale Harbor zuerst in seinen Cache. Das Image wird **direkt lokal ausgeliefert**, ohne das Netzwerk zum Management-Cluster zu belasten.

---
---
---
# 4.0 Bare-Metal Architektur: Sidero Omni & GitOps Pull-Modell

Diese Architektur beschreibt ein physisch getrenntes Bare-Metal-Setup. Sie trennt konsequent zwischen der **Verwaltungsebene (Hub / Management-Cluster)** und der **Ausführungsebene (Spokes / Ziel-Cluster)**.

Das Setup nutzt **Talos Linux** als API-gesteuertes, unveränderliches Betriebssystem auf den Bare-Metal-Servern. Die Hardware- und OS-Ebene wird zentral von **Sidero Omni** orchestriert, während das Applikations-Deployment über ein **GitOps Pull-Modell** via Argo CD erfolgt.

## 4.1 Das Management-Cluster (Der Hub / Die Kommandozentrale)

Das Management-Cluster läuft auf **separater, dedizierter physischer Hardware** (Bare-Metal). Es ist von den Workload-Clustern vollständig physisch und logisch getrennt. Es dient als "Seed-Cluster" und hostet die gesamte Control Plane für Infrastruktur und Applikationen.

**Basis-Schicht:**
* **Hardware:** Physisch getrennte Bare-Metal-Server.
* **Betriebssystem & K8s:** Ein eigenständiges Kubernetes-Cluster das stabil und hochverfügbar als Fundament läuft.

**Infrastruktur-Control-Plane (Die Server-Schmiede):**
* **Sidero Omni:** Die zentrale Instanz. Omni übernimmt das PXE-Booting, installiert Talos und ordnet die physischen Server logischen Clustern zu (z.B. "Test" oder "Prod").
* **Sidero Image Factory (Optional):** Baut auf dem Management-Cluster maßgeschneiderte Talos-OS-Images.

**Applikations-Control-Plane (Die Software-Fabrik):**
* **Gitea:** Quellcode-Repository (Single Source of Truth für den Code).
* **Tekton:** Führt CI-Pipelines aus. Baut Container-Images und verpackt Kubernetes-Manifeste zu OCI-Artefakten.
* **Master-Harbor:** Die zentrale OCI-Registry. Speichert alle Builds sicher ab.
* **Kargo:** Das CD-Logistik-Tool. Steuert die Release-Freigaben (Promotions) rein durch das Taggen von Paketen im Master-Harbor.
* **Argo CD (Global):** Steuert die Tools auf dem Management-Cluster selbst und installiert bei neuen Ziel-Clustern initial das *lokale* Argo CD.

---
---

## 4.2 Die Ziel-Cluster (Die Spokes / Test, Integra, Prod)

Die Ziel-Cluster hosten die eigentlichen Business-Applikationen. Sie laufen auf **anderen physischen Bare-Metal-Servern** als das Management-Cluster. Sie sind "dumm", hochsicher und kennen weder den Quellcode noch die CI-Pipelines.

**Basis-Schicht (Managed by Omni):**
* **Hardware:** Physisch getrennte Bare-Metal-Server.
* **Betriebssystem (Talos Linux):** Wird von Omni vollautomatisch über das Netzwerk installiert. Es ist Read-Only, hat keine SSH-Zugänge und wird rein über die Talos-API gesteuert.
* **Kubernetes:** Wird von Omni konfiguriert. Die Cluster sind via WireGuard an die zentrale Omni-Instanz auf dem Management-Cluster angebunden (für Health-Checks und Updates).

**Applikations-Ebene (GitOps Pull-Modell):**
*(Auf diesen Clustern laufen absichtlich KEIN Gitea, KEIN Tekton, KEIN Kargo und KEIN Omni)*
* **Lokales Argo CD:** Der autonome Agent auf dem Cluster. Er überwacht nur den lokalen Harbor-Proxy und zieht (pullt) freigegebene Pakete.
* **Lokaler Harbor (als Proxy Cache):** Ein schlanker Zwischenspeicher. Er lädt benötigte Container-Images und OCI-Pakete im Hintergrund einmalig vom *Master-Harbor* (Hub) herunter und puffert sie lokal. Dies spart massiv Netzwerktraffic und garantiert, dass das Cluster auch bei einem Ausfall des Management-Clusters autark Pods neu starten kann.
* **Business-Applikationen:** (z.B. `backend-api` in der jeweiligen Stage).

---
---
---
# 5.0 Gesamtarchitektur und Workflow-Zusammenfassung

Dieses Kapitel fasst die finale Architektur zusammen, welche die Provisionierung physischer Hardware (Bare-Metal) mit einem hochskalierbaren GitOps-Pull-Modell verbindet. Es wird streng zwischen einer zentralen Verwaltungsebene und einer dezentralen Ausführungsebene unterschieden.

## 5.1 Architektur-Übersicht: Hub and Spoke

Die Infrastruktur basiert auf einer physisch und logisch getrennten Architektur. Das Fundament auf den Bare-Metal-Servern bildet **Talos Linux**, ein API-gesteuertes und unveränderliches Betriebssystem für Kubernetes. 

### 5.1.1 Der Hub (Management-Cluster / "Die Kommandozentrale")
Das Management-Cluster läuft auf isolierter, dedizierter Bare-Metal-Hardware und dient als "Seed-Cluster" für die gesamte Umgebung. Es hostet alle Tools, die zur Erschaffung von Infrastruktur und zur Kompilierung von Code benötigt werden.

#### Infrastruktur-Ebene
* **Sidero Omni:** Die Control-Plane für Hardware. Fängt "Call-Homes" von leeren Bare-Metal-Servern ab, installiert Talos Linux und Kubernetes und weist die physischen Server logischen Clustern zu.
* *(Optional)* **Sidero Image Factory:** Generiert maßgeschneiderte Talos-ISO-Images mit spezifischen Hardware-Treibern.

#### Applikations-Ebene (GitOps-Pipeline)
* **Gitea:** Das GitOps-Management-Repository (Single Source of Truth).
* **Tekton:** Führt die CI-Prozesse aus, baut Images und verpackt die Konfiguration zu OCI-Artefakten.
* **Master-Harbor:** Die zentrale OCI-Registry, die alle gebauten Pakete speichert und scannt.
* **Kargo:** Das CD-Orchestrierungs-Tool, das die OCI-Pakete durch Tagging für die jeweiligen Stages (Test, Integra, Prod) freigibt.
* **Argo CD (Global):** Steuert die internen Tools des Hubs und installiert initial die benötigten Basis-Agenten auf neu erschaffenen Ziel-Clustern.

---

### 5.1.2 Die Spokes (Ziel-Cluster wie Test, Integra, Prod)
Die Ziel-Cluster laufen auf eigenen physischen Servern und hosten ausschließlich Business-Workloads. Sie agieren nach dem **GitOps Pull-Modell** und sind architektonisch verschlankt, wodurch sie hochsicher und ressourceneffizient sind.

#### Infrastruktur-Ebene
* **Talos Linux & Kubernetes:** Read-Only Betriebssystem, gesteuert über API und via WireGuard mit dem Omni-Hub verbunden.

#### Applikations-Ebene (Die Agenten)
*Auf diesen Clustern laufen absichtlich KEIN Gitea, KEIN Tekton, KEIN Kargo und KEIN Omni.*
* **Lokales Argo CD:** Der autonome Pull-Agent, der den lokalen Harbor überwacht und Änderungen direkt auf dem Cluster anwendet.
* **Lokaler Harbor (Proxy Cache):** Ein "dummer" Zwischenspeicher ohne Nutzerverwaltung. Er verweist auf den Master-Harbor, lädt einmalig benötigte Images/Pakete herunter und puffert diese lokal. Dies garantiert, dass das Spoke-Cluster auch bei Verbindungsabbruch zum Hub voll funktionsfähig bleibt.

---
---
## 5.2 Der End-to-End Workflow

Dieser Ablauf beschreibt den gesamten Lebenszyklus einer Änderung – von der physischen Bereitstellung bis zum Deployment in Produktion.

### 5.2.1 Phase 1: Hardware-Provisionierung (Omni)
1. **Boot:** Ein neuer physischer Server wird ins Rack geschoben und per ISO gestartet.
2. **Call-Home:** Der Server meldet sich bei **Sidero Omni** auf dem Hub.
3. **Zuweisung:** Der Server wird in der Omni-UI einem Ziel-Cluster (z.B. "Cluster-Prod") zugewiesen. Omni installiert Talos Linux und Kubernetes.
4. **Bootstrapping:** Das globale **Argo CD** auf dem Hub erkennt das neue Cluster und installiert vollautomatisch das *lokale Argo CD* sowie den *lokalen Harbor-Proxy* auf dem Spoke. Das Cluster ist nun einsatzbereit.

### 5.2.2 Phase 2: Applikations-Deployment (GitOps)
1. **Commit & Build (Hub):** Ein Entwickler committet Code. **Tekton** baut das Container-Image sowie das OCI-Paket und pusht es in den zentralen **Master-Harbor**.
2. **Promotion (Hub):** Ein Release-Manager gibt in **Kargo** die Version für die Produktion frei (das OCI-Paket erhält den Tag `prod`).
3. **Pull & Cache (Spoke):** Das **lokale Argo CD** auf dem Produktions-Cluster sieht das Update. Der **lokale Harbor-Proxy** lädt das Image vom Master-Harbor herunter und speichert es lokal.
4. **Execution (Spoke):** Das lokale Argo CD wendet die Kubernetes-Manifeste an. Die Workload-Pods starten und ziehen das Image schnell und netzwerkschonend aus dem lokalen Proxy-Cache.