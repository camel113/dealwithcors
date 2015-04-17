#Tentative d'explication du méchanisme CORS

Documentation [HTTP access control (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS)

##Simple request

Lorsque je fais une requête AJAX simple (Simple requests) depuis mon browser `http:localhost:8888` vers un service JSON distant `http:localhost:3000`.

> Rappel: A simple cross-site request is one that:
> * Only uses GET, HEAD or POST. If POST is used to send data to the server, the Content-Type of the data sent to the server with the HTTP POST request is one of application/x-www-form-urlencoded, multipart/form-data, or text/plain.
> * Does not set custom headers with the HTTP Request (such as X-Modified, etc.)

```javascript
$.ajax({
   url: "http://localhost:3000",
   type: "GET",
   success: function() { console.log('success'); }
});
```

Si le serveur `http:localhost:3000` a sa configuration CORS permettant un accès à ses ressources depuis `http:localhost:8888` alors le client reçoit en réponse à sa requête le document demandé.
![simple cors request debugbar image](https://github.com/camel113/dealwithcors/blob/master/images/simpleCORSrequest.png "simple cors request")
> On constate que le serveur autorise les requêtes provenant de l'origine `http:localhost:8888`

##Preflighted requests

Les requêtes sont prefilghted lorsque

> * It uses methods other than GET, HEAD or POST.  Also, if POST is used to send request data with a Content-Type other than application/x-www-form-urlencoded, multipart/form-data, or text/plain, e.g. if the POST request sends an XML payload to the server using application/xml or text/xml, then the request is preflighted.
> * It sets custom headers in the request (e.g. the request uses a header such as X-PINGOTHER)

Si je tente la requête suivante sur le serveur `http:localhost:3000`

```javascript
$.ajax({
    url: "http://localhost:3000",
    type: "GET",
    beforeSend: function(xhr){xhr.setRequestHeader('X-Requested-With', 'XMLHttpRequest');},
    success: function() { console.log('success'); }
 });
 ```
Une requête preflighted de type HTTP OPTIONS est d'abord envoyé par le browsers afin de savoir si le serveur accepte ce type de custom header `'X-Requested-With', 'XMLHttpRequest'`.
 
Si le serveur n'est pas configuré pour ce type d'header alors il répondra de cette manière là
 
> preflightedCORSerror.png
> On constate que le browser a envoyé une requête de type HTTP OPTIONS. La réponse n'est pas forcémment claire et n'indique pas clairement qu'il y a une erreur. Toutefois notre requête GET n'a pas pu être exécutée et le corps de la réponse est vide.

Configurons maintenant serveur node `http:localhost:3000` afin qu'il accepte les custom headers `X-Requested-With`

```javascript
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "http://localhost:8888");
  res.header("Access-Control-Allow-Headers", "x-requested-with");
  next();
});
 ```
Si on tente la même requête on a cette fois une autre réponse

> preflightedCORSok.png
> D'abord le serveur répond à la requête prefligthed par le browser. 
> Dans notre cas il autorise les requêtes avec headers `X-Requested-With`

Puis notre requête GET est executée et retourne la réponse souhaitée

> preflightedCORSget.png

## Pourquoi la requête éxecutée depuis notre code openlayers2 n'aboutit pas?

```javascript
vectorLyr = new OpenLayers.Layer.Vector("Vector layer from GeoJSON", {
  protocol: new OpenLayers.Protocol.HTTP({
      url: "http://tifelek02.heig-vd.ch/2.5_day.geojson",
      format: new OpenLayers.Format.GeoJSON()
  }),
  strategies: [new OpenLayers.Strategy.Fixed()],
});
 ```
Si on observe la requête qui part, il y a effectivement un custom headers de type `X-Requested-With` indépendamment de notre volonté.

> ol2CORSerror.png
> On constate que le serveur n'autorise pas explicitement les headers de type `X-Requested-With`. C'est pourquoi il ne retourne rien.