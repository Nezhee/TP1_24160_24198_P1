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