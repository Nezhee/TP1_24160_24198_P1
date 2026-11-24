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

  ## Étape 5 — Générer le code depuis le WSDL

**1.** Dans le WSDL :
- l'opération est `emprunterLivre` (dans `<wsdl:portType name="EmpruntPortType">`) ;
- son message de fault est `LivreIndisponibleFault`, qui contient l'élément
  `bib:LivreIndisponible` (isbn + retourPrevu) ;
- l'action SOAP à envoyer est `http://supnum.mr/bib/emprunterLivre`
  (dans `<soap:operation soapAction="..."/>` du binding).

**2.** Signature générée dans `EmpruntPortType` :
`public EmpruntType emprunterLivre(String idAdherent, String isbn) throws LivreIndisponibleFault`
avec `@WebMethod(action = "http://supnum.mr/bib/emprunterLivre")` et
`@WebResult(name = "emprunt")`.

**3.** `dateRetour` est généré en `XMLGregorianCalendar`, car le WSDL déclare
`xs:date`. Dans la partie A, c'était un simple `String` (`xs:string`) :
le contrat ne disait pas que c'était une date.

**4.** La classe d'exception générée s'appelle `LivreIndisponibleFault`.
Ce nom vient du `<wsdl:message name="LivreIndisponibleFault">` du WSDL.
Le contenu de l'erreur (isbn, retourPrevu) est dans une autre classe,
`LivreIndisponible`, qui vient de l'élément `<xs:element name="LivreIndisponible">`.

## Étape 6 — Implémenter et comparer

**1.** Oui, avec `wsdlLocation`, le WSDL publié est exactement celui que
nous avons fourni (seulement le commentaire « Published by », pas
« Generated by »). Il contient bien le motif de l'ISBN
(`<xs:pattern value="97[89]-\d-\d{2,5}-\d{2,6}-\d"/>`) et le type
`xs:date` pour `dateRetour` et `retourPrevu`.

**2.** Oui. Avec `01-emprunter.xml` tel quel, on obtient une Fault
`S:Server` : « Cannot invoke "String.equals(Object)" because "isbn" is null ».
Le WSDL de la partie B a `elementFormDefault="qualified"` : les paramètres
doivent être dans le namespace `bib`. Dans `01`, `<idAdherent>` et `<isbn>`
n'ont pas de préfixe, donc ils sont ignorés et arrivent à `null`.
Nous avons créé `xml/05-emprunter-contractfirst.xml` avec
`<bib:idAdherent>` et `<bib:isbn>` : la réponse est alors HTTP 200
(`E-2026-0812`, `2026-12-08`). Avec l'ISBN `978-2-10-074444-4`, on reçoit
la Fault `S:Server` « Livre indisponible » avec le `<detail>`
`LivreIndisponible`.

**3.** Sans `@SchemaValidation`, l'ISBN `978-2-10` est **accepté**
(HTTP 200) : par défaut, JAX-WS ne vérifie pas le XML reçu par rapport au
schéma. Avec `@SchemaValidation`, il est refusé avant d'arriver au code
Java : Fault `S:Client`, « cvc-pattern-valid: Value '978-2-10' is not
facet-valid with respect to pattern '97[89]-\d-\d{2,5}-\d{2,6}-\d' for
type 'IsbnType' ».

**4.** Sans `wsdlLocation`, JAX-WS ne publie plus notre fichier : il
**régénère** un WSDL à partir des classes Java (commentaire « Generated by »).
Les types sont déplacés dans `?xsd=1`, les noms changent (binding
`EmpruntPortBinding` au lieu de `EmpruntSoapBinding`, message
`emprunterLivre` au lieu de `emprunterLivreRequest`, fault
`LivreIndisponibleFault`) et des `wsam:Action` sont ajoutés. Surtout,
`isbn` devient un simple `xs:string` : **le motif de l'ISBN est perdu**.
`xs:date` et `elementFormDefault="qualified"` restent, car ils sont
dans les annotations des classes générées. Conséquence : même avec
`@SchemaValidation`, l'ISBN `978-2-10` est de nouveau accepté (HTTP 200).

## Conclusion — Choisir et rendre

**1.** Deux différences entre le WSDL de la partie A et celui de la partie B :
- En A, `isbn` et `dateRetour` sont des `xs:string` (pas de règle).
  En B, `isbn` a un type `IsbnType` avec un `pattern`, et les dates
  sont en `xs:date`.
- En A, les paramètres sont "unqualified" (sans préfixe) et
  `soapAction=""`. En B, `elementFormDefault="qualified"` (paramètres
  avec `bib:`) et `soapAction="http://supnum.mr/bib/emprunterLivre"`.

**2.** C'est le projet **code-first (partie A)** qui voit son contrat
changer, car son WSDL est généré automatiquement à partir du code Java.
Le risque : les clients déjà construits avec l'ancien WSDL envoient ou
attendent encore `dateRetour`. L'élément ne correspond plus, il est
ignoré ou provoque une erreur, et les clients cassent sans prévenir.
En contract-first, le WSDL est fixe : le code ne peut pas changer le contrat.

**3.** Pour le service **Paiement**, appelé par Scolarité (.NET),
Bibliothèque (PHP) et le portail (Node.js), nous choisirions
**contract-first** : plusieurs équipes et plusieurs langages doivent
partager un contrat stable, écrit et validé à l'avance.
Pour un **prototype interne de deux jours**, nous choisirions
**code-first** : c'est plus rapide et il n'y a qu'un seul consommateur.
Il faudra simplement figer le WSDL avant une mise en production.
