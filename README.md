# REDAXO 5 URL2 Fork

## Beschreibung

Fork des beliebten REDAXO 5 AddOns zur URL-Generierung.

## Installation

* Via Install AddOn im Backend herunterladen
* AddOn installieren und aktivieren

### unterstützte Rewriter

* [yrewrite](https://github.com/yakamara/redaxo_yrewrite)

## Beispiel: Filme

Normalerweise wird ein Film über eine Url wie `/filme/?movie_id=1` geholt.

Mit dem AddOn ist es möglich Urls wie `/filme/the-big-lebowski/` zu erzeugen.

Der REDAXO Artikel `/the-big-lebowski/` selbst existiert dabei nicht. Es wird alles im REDAXO Artikel `/filme/` abgehandelt.
**The Big Lebowski** ist dabei der Titel eines Filmes, welcher in einer eigenen Datenbanktabelle hinterlegt wurde. 


### Url holen 
Um die Url eines einzelnen Filmes auszugeben verwendet man:

```php
echo rex_getUrl('', '', ['movie-id' => $movieId]);
```

| Variable         | Beschreibung                 |
| ---------------- | ---------------------------- |
| `movie-id` | ist der im Profil hinterlegte Namensraum |
| `$movieId`    | Datensatz Id des Filmes |


### Id holen 
Nach dem der Film mit der eigenen Url aufgerufen wurde (Darstellung der Detailseite), muss jetzt die dazugehörige Datensatz-Id ermittelt werden. Erst dann können die eigentlichen Daten aus der Tabelle abgerufen und ausgegeben werden.

```php
// versuche die Url aufzulösen
$manager = Url\Url::resolveCurrent();
$movieId = $manager->getDatasetId();
```

### Beispiel Code

```php
<?php
use Url\Url;

$manager = Url::resolveCurrent();

if ($manager) {
    $movie = rex_yform_manager_table::get('rex_movie')->query()->findId($manager->getDatasetId());
    // oder wenn ModelClass genutzt wird
    $movie = Movie::get($manager->getDatasetId());
    if ($movie) {
        dump($movie);
    }
} else {
    $movies = rex_yform_manager_table::get('rex_movie')->query()->find();
    // oder wenn ModelClass genutzt wird
    $movies = Movie::getAll();
    if (count($movies)) {
        foreach ($movies as $movie) {
            echo '<a href="' . rex_getUrl('', '', ['movie-id' => $movie->getId()]) . '">' . $movie->getValue('title') . '</a>';
        }
    }
}
```

### Zusätzliche Pfade aus Relationen bilden

Möchte man den Filmen Genres zuordnen, passiert dies meist über eine Relation zu diesen Kategorien.
Die Urls dazu könnten dann so aussehen: `/filme/komoedie/the-big-lebowski/`
 

### Zusätzliche Pfade für die Url

#### eigene Pfade an die Url hängen

Im Feld **eigene Pfade an die Url hängen** lassen sich zusätzliche Pfade eintragen, die als gültige Urls verwendet werden können. So ließe sich beispielsweise bei einem Film noch eine zusätzliche Seite über `/filme/the-big-lebowski/zitate/` anzeigen. Dann muss in dem Textfeld einfach nur `zitate` eingetragen werden - ohne vorangestellten und abschließenden Schrägstrich.

<del>Bei der Ausgabe kann man dann `$mypath = UrlGenerator::getCurrentOwnPath();` schreiben. Wenn die Seite mit dem Zusatz /info aufgerufen wird, enthält `$mypath` den Wert `info`.</del>


#### Unterkategorien anhängen?

Es können an die erzeugte Url auch die Unterkategorien des Artikels angehangen werden
Die Urls dazu könnten dann so aussehen: `/filme/the-big-lebowski/schauspieler/`

`Schauspieler` ist dabei eine Unterkategorie der Kategorie `Filme` innerhalb der Strukturverwaltung.

### Beispiel: URL-Pathlist neu generieren

Wenn Datenbanktabellen außerhalb des YForm-Table-Managers befüllt werden, greift der passende EP nicht und die URLs werden nicht neu generiert. Dies lässt sich im Code nachholen, indem folgender Code verwendet wird.

```php
$profiles = \Url\Profile::getAll();
if ($profiles) {
	foreach ($profiles as $profile) {
		$profile->deleteUrls();
		$profile->buildUrls();
	}
}
```

## Extension Points

Der Extension Point URL_MANAGER_PRE_SAVE gibt die Möglichkeit eine URL vor dem Speichern in der URL Tabelle zumanipulieren.

### Beispiel Code URL_MANAGER_PRE_SAVE

```php
<?php
rex_extension::register('URL_MANAGER_PRE_SAVE', 'rex_url_shortener');

/**
 * Kürzt URL für ALLE Profile indem es die Artikel und Kategorienamen aus der URL entfernt.
 * @param rex_extension_point $ep Redaxo extension point
 * @return Url Neue URL
 */
function rex_url_shortener(rex_extension_point $ep) {
	$params = $ep->getParams();
	$url = $params['object'];
	$article_id = $params['article_id'];
	$clang_id = $params['clang_id'];
	
	// URL muss nur gekürzt werden, wenn es sich nicht im den Startartikel der Domain handelt
	if($article_id != rex_yrewrite::getDomainByArticleId($article_id, $clang_id)->getStartId()) {
		$article_url = rex_getUrl($article_id, $clang_id);
		$start_article_url = rex_getUrl(rex_yrewrite::getDomainByArticleId($article_id, $clang_id)->getStartId(), $clang_id);
		$article_url_without_lang_slug = '';
		if(strlen($start_article_url) == 1) {
            		// Wenn lang slug  im Startartikel nicht angezeigt wird
			$article_url_without_lang_slug = str_replace('/'. strtolower(rex_clang::get($clang_id)->getCode()) .'/', '/', $article_url);
		}
		else {
			$article_url_without_lang_slug = str_replace($start_article_url, '/', $article_url);
		}
		
		// Im Fall $url ist urlencoded, muss Artikel URL ebenfalls encoded werden
		$article_url_without_lang_slug_split = explode("/", $article_url_without_lang_slug);
		for($i = 0; $i < count($article_url_without_lang_slug_split); $i++) {
			$article_url_without_lang_slug_split[$i] = urlencode($article_url_without_lang_slug_split[$i]);
		}
		$article_url_without_lang_slug_split_encoded = implode("/", $article_url_without_lang_slug_split);

		$new_url = new \Url\Url(str_replace($$article_url_without_lang_slug_split_encoded, '/', $url->__toString()));
		
		// Auf Duplikate prüfen
		$query = "SELECT * FROM ". \rex::getTablePrefix() ."url_generator_url "
			."WHERE url = '". $new_url->__toString() ."'";

		$result = \rex_sql::factory();
		$result->setQuery($query);
		if($result->getRows() > 0) {
			// FALSE zurückgeben, Duplikate sind nicht erlaubt
			return FALSE;
		}

		return $new_url;
	}
	
	return $url;
}
```

## SEO-Methoden
Für die ordnungsgemäße Ausgabe müssen die YRewrite-Tags für Titel, Beschreibung u.a. durch die SEO-Klasse des URL-Addons ausgetauscht werden.  `$seo = new rex_yrewrite_seo();` wird ersetzt durch `$urlSeo = new Url\Seo();`:

```php
$urlSeo = new Url\Seo();
    echo $urlSeo->getTitleTag().PHP_EOL;
    echo $urlSeo->getDescriptionTag().PHP_EOL;
    echo $urlSeo->getRobotsTag().PHP_EOL;
    echo $urlSeo->getHreflangTags().PHP_EOL;
    echo $urlSeo->getCanonicalUrlTag().PHP_EOL;
```

## Weitere Tipps 

### Leere Einträge vermeiden

Werden URLs selbst erzeugt, z.B. über eine YForm-Tabelle, dann darf das Feld oder die Feldkombination, aus der die URL generiert wird, nur einmal vorkommen (prüfen auf `unique`) und außerdem niemals leer sein (prüfen auf `empty`).

### Von Url generierte Seite mit YForm-Formular

Befindet sich ein Formular auf einer Seite, die über eine URL des URL-Addon aufgerufen wurde, so muss die Ziel-URL des Formulars angepasst werden. 

```php
$yform->setObjectparams('form_action',rex_getUrl('', '', [$data->urlParamKey => $id]));
``` 

oder

```php
$yform->setObjectparams('form_action',Url::current());
``` 

Weiere Infos zu den Objekt-Parametern von YForm befinden sich in der YForm-Doku.
