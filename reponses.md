# TP1 — SOAP et WSDL

Nezhe Moulaye Ahmed — 24160
Fatimetou El Ghalya Kemal — 24198

## Étape 1 — Créer et lancer le service

**1.** L'annotation `@WebService` fait de `EmpruntService` un service web.
L'annotation `@WebFault` fait de `LivreIndisponibleException` une erreur
déclarée dans le contrat (WSDL).

**2.** Les classes de données doivent avoir un constructeur vide car
JAX-WS (via JAXB) doit d'abord créer l'objet vide, puis le remplir avec
les valeurs lues dans le XML. Sans constructeur vide, il ne peut pas
transformer le XML en objet Java (ni l'inverse).

## Étape 2 — Lire le WSDL généré

**1.** URL du service (dans `<service>` → `soap:address`) :
`http://localhost:8080/bib/EmpruntService`

Deux opérations (dans `<portType>`) :
- `emprunterLivre` : entrée `emprunterLivre`, sortie `emprunterLivreResponse`,
  et un fault `LivreIndisponibleException`.
- `rendreLivre` : entrée `rendreLivre`, sortie `rendreLivreResponse`.

**2.** Le binding déclare `style="document"` et `use="literal"`
(document/literal). C'est du **SOAP 1.1**, car le WSDL utilise
l'espace de noms `http://schemas.xmlsoap.org/wsdl/soap/` et le transport
`http://schemas.xmlsoap.org/soap/http`. Pour SOAP 1.2, on aurait `soap12`.

**3.** Dans le XSD (`?xsd=1`), `isbn` et `dateRetour` sont générés en
`xs:string`, car ce sont des `String` en Java. La règle de format de
l'ISBN (`97[89]-\d-...`) n'apparaît **pas** : elle est seulement dans le
code Java (`Pattern`), JAX-WS ne peut pas la mettre dans le contrat.
Le client ne la connaît donc pas avant d'envoyer sa requête.

**4.** Nous avons utilisé curl à la place de SoapUI. Dans notre requête,
`idAdherent` et `isbn` n'ont **pas** de préfixe d'espace de noms,
seul l'élément `bib:emprunterLivre` en a un. C'est parce que le XSD
généré n'a pas `elementFormDefault="qualified"` : les éléments enfants
sont donc "unqualified" (sans namespace).

**5.** Avec `curl -v`, on voit les trois couches :
- **HTTP** : les lignes `POST /bib/EmpruntService`, `Content-Type: text/xml`,
  `SOAPAction` et la réponse `HTTP/1.1 200`. HTTP sert à **transporter**
  le message.
- **XML** : le contenu envoyé et reçu. XML est le **format** des données.
- **SOAP** : la structure `Envelope` / `Header` / `Body`. SOAP définit
  **comment organiser** le message (et les erreurs avec Fault).
- **WSDL** : il n'apparaît pas dans le message. C'est le **contrat** qui
  décrit le service (URL, opérations, paramètres). On l'a lu pour savoir
  comment écrire notre requête.

## Étape 3 — Envoyer des messages SOAP

**1.** La requête utilise toujours `POST` car SOAP envoie une enveloppe
XML dans le corps de la requête, et une requête GET n'a pas de corps.
Avec SOAP, toutes les opérations passent par la même URL : c'est le
contenu du `Body` (le nom de l'opération) qui dit quoi faire, pas la
méthode HTTP.

**2.** Non, le bloc `jeton` n'a pas empêché le traitement (réponse
HTTP 200 avec le numéro `E-2026-0812`). Le service ne connaît pas ce
bloc, mais comme il n'a pas `mustUnderstand="1"`, il est simplement ignoré.

**3.** Le service répond HTTP 500 avec une Fault
`faultcode = S:VersionMismatch` : « Attente de Envelope dans l'espace de
noms http://schemas.xmlsoap.org/soap/envelope/, mais
http://www.w3.org/2003/05/soap-envelope obtenu ».
C'est logique : le binding lu à l'étape 2 est en SOAP 1.1
(`http://schemas.xmlsoap.org/wsdl/soap/`), donc le service n'accepte que
l'enveloppe SOAP 1.1. On remarque que la Fault elle-même est en SOAP 1.1.

**4.** Les cinq erreurs et leur correction (fichier `xml/04-rendre.xml`) :
1. `soap:envelope` → `soap:Envelope` (XML est sensible à la casse).
2. Namespace `http://supnum.mr/bibliotheque` → `http://supnum.mr/bib`
   (sinon le service ne trouve pas l'opération).
3. `<bib:numero>` → `<numeroEmprunt>` : mauvais nom de paramètre, et il
   ne faut pas de préfixe (éléments enfants "unqualified").
4. `</rendreLivre>` → `</bib:rendreLivre>` : la balise de fermeture doit
   être identique à l'ouverture, sinon le XML est mal formé.
5. Le `Header` est après le `Body` → il doit être placé **avant** le Body.

L'erreur sur le paramètre (n°3) ne déclenchait aucune erreur côté
serveur (avant la modification de l'étape 4) : l'élément inconnu est
simplement ignoré, `numeroEmprunt` arrive à `null` et le service
renvoyait quand même `0`. C'est dangereux car l'erreur passe inaperçue.

Remarque : avec `mvn exec:java`, les Faults revenaient vides (SAAJ/JAXB
introuvables dans le classloader de Maven). Nous avons lancé `Serveur`
directement depuis IntelliJ pour corriger ce problème.

## Étape 4 — Provoquer et construire des Faults

**1.** Cas 1 (ISBN `978-2-10`) : `faultcode` = `S:Client`,
`faultstring` = `ISBN mal formé : 978-2-10`, code HTTP **500**.
Cas 2 (ISBN `978-2-10-074444-4`) : `faultcode` = `S:Server`,
`faultstring` = `Livre indisponible`, code HTTP **500**.

**2.** Dans le cas 2, `<detail>` contient l'élément `LivreIndisponible`
avec `<isbn>978-2-10-074444-4</isbn>` et `<retourPrevu>2026-12-02</retourPrevu>`.
Sa structure est décrite dans le WSDL : dans le XSD (élément
`LivreIndisponible` de type `livreIndisponibleInfo`), dans
`<message name="LivreIndisponibleException">` et dans le `<fault>` de
l'opération `emprunterLivre`.

**3.** JAX-WS a choisi le code `Server`. Mais ce n'est pas une erreur du
serveur (il fonctionne bien) : c'est la demande du client qui ne peut pas
être satisfaite. Nous aurions choisi `Client` (`Sender` en SOAP 1.2),
car renvoyer la même requête donnera toujours la même erreur.

**4.** Avec `mustUnderstand="1"`, le service refuse le message :
`faultcode = S:MustUnderstand`, « MustUnderstand headers:[{http://supnum.mr/bib}jeton]
are not understood » (HTTP 500). Le service est obligé de comprendre le
bloc `jeton`, mais il ne le connaît pas. Avec `"0"` (ou sans l'attribut),
le bloc est facultatif : il est ignoré et la requête est traitée normalement.

Pour `rendreLivre` avec le numéro `X-999`, on obtient bien
`faultcode = S:Client`, « Numéro d'emprunt inconnu : X-999 ».

**5.** Différences entre notre Fault 1.2 (`xml/03-fault-sender-12.xml`)
et les Faults 1.1 reçues :
- 1.1 : `faultcode`, `faultstring`, `detail` (éléments sans namespace).
- 1.2 : `Code/Value`, `Reason/Text`, `Detail`, dans le namespace SOAP 1.2.
- Les codes changent : `Client` → `Sender`, `Server` → `Receiver`.
- En 1.2, `Reason/Text` a un attribut `xml:lang` : le message peut être
  donné en plusieurs langues.