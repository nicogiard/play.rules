# Services Web

## REST et RESTful, qu'est ce que c'est?

On appelle service RESTful un service web respectant le style d'architecture REST.
REST (Representational State Transfer) est un modèle d'architecture orienté ressources. Ceci signifie qu'au lieu d'exposer des méthodes comme lorsque l'on utilise le protocole SOAP, on va exposer des ressources. Chaque ressource possède une URL qui l'identifie.

Contrairement à SOAP, REST s'appuie uniquement sur le protocole HTTP et ne propose aucune couche au dessus de ce protocole. Tout est faisable à partir des opérations fournies par de base par HTTP : GET, PUT, POST, DELETE, etc.
Pour récupérer une collection, on effectue un GET sur l'URL appropriée.
La réponse contiendra un ensemble d'éléments, décrits par exemple en XML ou en JSON. Pour chaque élément, une URL est définie. Il sera donc possible d'effectuer un appel GET sur un élément en particulier pour ne récupérer que celui ci. Une opération de type PUT sur le même élément permettra de mettre à jour ses données. De la même façon, une opération DELETE supprimera l'élément.

REST est en fait le modèle sur lequel le web lui même est construit : les sites et les pages web étant des ressources accessibles via des URL, depuis un navigateur grâce à des opérations HTTP.

Pour la sécurité il est possible de s'appuyer sur l'authentification HTTP, ou encore sur le SSL avec HTTPS. Comme vous pouvez le voir, tout est fait pour utiliser au maximum ce que le web nous fournit depuis toujours, sans sur-couche supplémentaire.

## Play!► et les services REST

Les URL de Play!► étant RESTful par essence, il devient très facile de créer une petite API REST/XML conjointement à l'interface Web d'une application Play!►.
Voyons comment procéder.

### Exposer des données avec un service REST

Gardons l'exemple de notre bibliothèque musicale. Notre modèle comporte des albums, des artistes et des genres.
Pour rappel, la classe Album se présente comme ceci :

~~~ java
@Entity
public class Album extends Model {
public String name;
@ManyToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
public Artist artist;
public Date releaseDate;
@Enumerated(EnumType.STRING)
public Genre genre;
~~~

Nous voulons définir une URL qui renvoie lors d'un GET la liste des albums au format XML pour un genre donné.
Pour cela nous devons modifier le fichier routes :

	GET /albums/{genre}       Application.list
	GET /api/albums/{genre}   Application.listXml(format:'xml')

La première ligne correspond à la page HTML(non présentée dans cet article) affichant la liste des albums disponibles : le format n'étant pas spécifié, le rendu se fera avec une page HTML.
Ici c'est la deuxième ligne qui nous intéresse. Le paramètre `(format:'xml')` indique que la méthode `render` du contrôleur devra chercher un fichier nommé listXml.xml.
Le paramètre {genre} sera récupéré dans l'URL et passé au contrôleur.

NB :
Il est possible d'utiliser une seule méthode dans le contrôleur si les paramètres requis et les traitements sont identiques pour les 2 types de rendus.
Dans notre cas il se peut qu'on ajoute des paramètres à la version HTML ultérieurement, sans vouloir impacter le rendu XML, par exemple :
	
	GET /albums/{genre}/{first}/{count} Application.list

J'ai donc opté pour une séparation du rendu dans deux méthodes distinctes.

Le code de la méthode Application.listXml est le suivant :

~~~ java
public static void listXml(String genre) {
        Genre genreEnum = Genre.valueOf(genre.toString().toUpperCase());
        List<Album> albums= Album.find("byGenre",genreEnum).fetch();
        render(albums);
}
~~~

Je recherche simplement les albums correspondant au genre passé en paramètre, et je demande le rendu de la liste. Au passage on voit la simplicité d'utilisation de JPA avec Play!►. Le rendu sera fait dans le fichier portant le nom de la méthode et l'extension xml : listXml.xml.
Ce template, placé dans le repertoire app/views, est défini comme ceci :

~~~ xml
<albums>
#{list albums, as:'album'}
    <album>
        <artist>${album.artist.name}</artist>
        <name>${album.name}</name>
        <release-date>${album.releaseDate.format('yyyy')}</release-date>
        <genre>${album.genre.toString()}</genre>
    </album>
#{/list}
</albums>
~~~


Voilà, cela suffit pour exposer nos albums en XML. En respectant le pattern d'URL défini dans le fichier routes, par exemple en appelant `http://localhost:9000/albums/rock`, on obtient le résultat suivant :

~~~ xml
<albums>
   <album>
      <artist>Nirvana</artist>
      <name>Nevermind</name>
      <release-date>1991</release-date>
      <genre>ROCK</genre>
   </album>
   <album>
      <artist>Muse</artist>
      <name>Origin of Symmetry</name>
      <release-date>2001</release-date>
      <genre>ROCK</genre>
      </album>
   <album>
      <artist>Muse</artist>
      <name>Black Holes and Revelations</name>
      <release-date>2006</release-date>
      <genre>ROCK</genre>
   </album>
</albums>
~~~

### Envoi de données à travers un service REST

Dans la première partie nous avons vu comment exposer des données au format XML avec Play!►.
Maintenant nous allons effectuer l'opération inverse, l'envoi d'un contenu XML au contrôleur à travers une URL RESTful.

On veut par exemple envoyer le contenu suivant en POST avec un content type application/xml :

~~~ xml
<album>
      <artist>Metallica</artist>
      <name>Death Magnetic</name>
      <release-date>2008</release-date>
      <genre>METAL</genre>
</album>
~~~


Pour cela on ajoute la ligne suivante au fichier routes pour autoriser l'opération POST sur l'url `/album`:

	POST /api/album  Application.saveXml

La méthode `saveXml` récupère le contenu de la requête dans la variable `request.body`.
Elle parse ensuite le contenu pour créer un album et l'enregistrer dans la base. La classe play.libs.XPath facilite le parcours de documents XML :

~~~ java
public static void saveXML(){
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    Document document = null;
    try{
    //création du document XML à partir de la requête
    DocumentBuilder builder = factory.newDocumentBuilder();
    document = builder.parse(request.body);
    }
    catch(Exception e){
    }
    //parsing du contenu XML
    Element albumNode = document.getDocumentElement();
	//artiste
	Node artistNode = XPath.selectNode("artist", albumNode);
	String artistName = XPath.selectText("name",artistNode);
	Artist artist = new Artist(artistName);
	//get the name
	String albumName = XPath.selectText("name", albumNode);
	Album album = new Album(albumName);
	//get the date
	String date = XPath.selectText("release-date",albumNode);
	DateFormat dateFormat = new SimpleDateFormat("yyyy");
	try {
		album.releaseDate = dateFormat.parse(date);
	} catch (ParseException e) {
		Logger.error(e.getMessage());
	}
	//genre
	String genre = XPath.selectText("genre", albumNode);
	Genre genreEnum = Genre.valueOf(genre.toString().toUpperCase());
	album.genre = genreEnum;

	//sauvegarde
	album.artist = artist;
	album.save();
}
~~~

NB: il est bien sûr possible d'obtenir un code moins verbeux en dé-sérialisant l'objet à l'aide d'un outil comme JAXB ou XStream, mais ce n'est pas l'objet de ce chapitre.

Lorsqu'on écrit le code album.artist=artist, la méthode `setArtist` est appelée automatiquement par Play!► (le code est modifié au runtime). On peut ainsi vérifier le fait que l'artiste existe ou non dans la base, pour savoir si on doit créer une nouvelle instance d'artiste ou récupérer l'artiste existant.
La méthode save() de la classe Album s'occupe alors d'enregistrer l'album en base, ainsi que l'artiste si il est inconnu dans la bibliothèque(à l'aide d'un cascade JPA).

~~~ java
public void setArtist(Artist artist){
    List<Artist> existingArtists = Artist.find("byName", artist.name).fetch();
    if(existingArtists.size()>0){
        //Le nom d'artiste est unique
        this.artist=existingArtists.get(0);
    }
    else{
        this.artist=artist;
    }
}
~~~

Notre API REST/XML nous permet donc maintenant de lire la liste des albums de note bibliothèque musicale et d'ajouter des albums.
Vous pouvez tester l'envoi de contenu XML avec l'application [rest-client](http://code.google.com/p/rest-client/) ou avec le plugin [Poster de Firefox](https://addons.mozilla.org/de/firefox/addon/poster/):

![Alt "poster"](https://static-cdn.addons.mozilla.net/img/uploads/previews/full/19/19951.png?modified=1208233459)

### Services REST/JSON

Dans la première partie de ce chapitre, nous avons vu comment créer des services REST envoyant et consommant des messages au format XML.
Voyons maintenant comment faire la même chose avec JSON.

#### Le format JSON

Définition de wikipedia : 

`JSON (JavaScript Object Notation) est un format de données textuel, générique, dérivé de la notation des objets du langage ECMAScript. Il permet de représenter de l’information structurée.`
	
L'avantage de JSON par rapport à XML être d'être un peu moins verbeux et directement interprétable dans un navigateur à l'aide de JavaScript.

Si on écrit cette ligne dans le fichier routes :

	GET /api/albums.json            Application.listAlbumsInJson  

Et cette méthode dans le contrôleur :

~~~ java 	
public static void listAlbumsInJson(){
        List<Album> albums = Album.findAll();
        renderJSON(albums);
}
~~~	 

L'appel de l'URL http://monappli/albums.json renverra directement notre liste d'objets albums au format JSON. Difficile de faire plus simple!

Autre astuce (que j'ai découvert grâce site zengularity.com) : pour déterminer directement le format de données à partir de l'URL, il est possible d'utiliser cette syntaxe dans le fichier routes :

	GET /api/albums.{<json|xml>format} Application.listAlbums  

En appelant /albums.xml , Play!► appellera la méthode `listAlbums` avec le paramètre 'format' initialisé à 'xml', et en appelant `/albums.json` ce même paramètre aura la valeur 'json'. 

On peut ensuite s'en servir dans le contrôleur : 

~~~ java 	
public static void listAlbums() {
    List<Album> albums = Album.all().fetch();
    if(request.format.equals("json"))
    renderJSON(albums);
    render(albums);
}
~~~ 
	  
Si vous tapez l'URL /albums.xml, Play!► cherchera un fichier de template XML nommé `listAlbums.xml` (une autre extension fonctionnerait aussi) pour effectuer le rendu.

### Recevoir un message JSON

Maintenant que nous savons exposer des données au format JSON à travers un service REST, voyons comment envoyer des données au serveur en utilisant le même format.
Cette méthode du contrôleur permet de résoudre cette problématique :

~~~ java 
public static void saveAlbumJson() {
    Gson gson = new Gson();
    Album album = gson.fromJson(new InputStreamReader(request.body),Album.class);
    album.replaceDuplicateArtist();
    album.save();
}
~~~ 

En récupérant l'objet `request.body`, on peut analyser le flux entrant et enregistrer un album dans la base de données.
Attention, pour que cette méthode fonctionne, il faudra respecter la structure de la classe Album lors de l'envoie des données en JSON. 

Si on veut fournir un point d'entrée unique pour enregistrer de nouveaux albums en XML ou JSON, on peut écrire une méthode qui se chargera de rediriger vers le bon traitement selon le _content type_ demandé dans la requête HTTP.

A la place de la ligne `POST /api/album  Application.saveXml` dans le fichier routes, on écrit :

	POST /api/album  Application.saveByApi

On ajoute ensuite cette méthode dans le contrôleur :

~~~java
public static void saveAlbumByApi() {
	if (request.contentType.equalsIgnoreCase("application/xml"))
		saveAlbumXml();
	else if (request.contentType.equalsIgnoreCase("application/json"))
		saveAlbumJson();
    }
~~~		

## Appeler un service externe avec Play!►.libs.WS

Play!► inclue également une libraire pour écrire des clients de services REST. Pour appeler un service externe, on peut écrire :

~~~java
import play.libs.WS;
//...
HttpResponse response = WS.url("http://api.twitter.com/1/statuses/user_timeline/loic_d.json").get();
~~~

On peut ensuite récupérer le résultat comme ceci : 

~~~java
JsonElement json = response.getJson();
~~~	

Cette librairie est aussi compatible avec les services renvoyant du XML ou du texte brut :

~~~java
String content = response.getString();
Document xml= response.getXml();
~~~