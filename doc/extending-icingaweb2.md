# Extending Icinga Web 2

## Writing your own Icinga Web Module

Welcome! Nice to see you're here to write your very first Icinga Web module. Icinga Web makes it as easy as possible. During the next hour, we'll show you with a help of series of examples how easy it is.

## Do i really need it? Why?

Of course, why not? It's wonderfully easy, and Icinga is a 100% free Open Source Software with a great community. Icinga Web 2 represents a stable, easy to understand and future-proof Platform. It is exactly what you need to build your own project from.

## Is it only for monitoring?

Not quite so! Of course, monitoring is what Icinga Web comes from. That's its strong point, where it feels at home. Since monitoring systems can communicate with all possible systems/appliances in and out of your Datacenter, we find it obvious that the front-end could do the same.

Icinga Web wants to be a modular framework, which aims to provide possibly easy integration of third-party software. We want to offer honest words of thanks for 3rd party projects - it's easy to use Icinga Logic with their own software.

Whether you want connection with 3rd party system, connection to a CMDB or complement classic monitoring with complex system visualization - it all can be done.


## But I don't know any PHP/JavaScript/HTML5

No problem. It's not a big problem to get started, no need for in-depth knowledge of web development to get going with writing your own modules.

# Preparation
For the purpose of this training, we'll use a basic Debian installation - mostly to show how little dependencies Icinga Web 2 has. While there are packages for all common distributions, for better learning effect we'll be working directly with GIT source tree.

To wake up our notebooks from weekend nap, let's start with a simple exercise:


    apt-get update

    # A few useful tools to get started
    apt-get install git vim wget

    # Icinga Web 2 dependencies:
    apt-get install php5-cli php5-mysql php5-gd php5-intl php5-ldap

    # up to date source code of Icinga Web 2 master branch:
    cd /usr/local
    git clone https://git.icinga.org/icingaweb2.git


While git does its thing, let's read up the training plan:

## Training plan

* Icinga Web 2 Architecture
* Preparing your own module
  * CLI commands
  * Working with parameters
  * Colors and other gimmicks
* Extending the Web Frontend
  * Adding pictures
  * Adding stylesheets
  * Extending menus
  * Preparing Dashboards
* Working with data
  * Preparation
  * Putting code in libraries
  * Working with parameters
  * Tricks for improving workflow
* Configuration
* Translations
* Integration with 3rd party software
* Free Lab


## Icinga Web 2 Architecture

During the development of Icinga Web 2 , the following things were the points of focus:

* Simplicity.
* Speed
* Reliability

We have already the DevOps movement described, our target user group for Icinga Web 2 are the administrators. Therefore we're trying to keep amount of external dependencies as low as possible. Therefore we give up on various hip features, to reduce likelihood of breakage due to upstream changes resulting in change of functionality. Best example is this training - prepared over the course of a year, long before first stable release of Icinga Web 2 - and yet it works just fine with almost all exercises without any changes required.

The web interface was designed so as to be able to present weekly or monthly problem overview on single screen, that could be e.g. put up on a wall mounted display. We wanted to be able to let us see there the state representing our environment. Any potential issues would be presented there, also when they stem from the application itself. Resolving the problem should keep things running as usual. And this without necessity of manual interaction.

## Used libraries

* Zend Framework 1.x
* jQuery 1.11 und 2.1
* Smaller PHP Libraries
  * HTMLPurifier
  * DOMPdf
  * lessphp
  * Parsedown
* Smaller JS libraries
  * jquery-...

## Anatomy of an Icinga Web 2 module


Icinga Web 2 follows the paradigm of "conventional configuration". After our experiences with Icinga Web 1, we came to conclusion, that the best tool for XML processing is `/bin/rm` . Who sticks to simpler conventions, spares himself a lot of configuration work. The basic assumption being that one needs to dig through configuration files only on special occasions with Icinga Web. It is usually suffices to just copy a file into a specific location, with minor edits.

A complex, mature module could contain the following structure:

    .
    └── training                Main directory of the module
        ├── application
        │   ├── clicommands     CLI commands
        │   ├── controllers     Web Controller
        │   ├── forms           Forms
        │   ├── locale          Translations
        │   └── views
        │       ├── helpers     View Helpers
        │       └── scripts     View Scripts
        ├── configuration.php   Preparation methods of Menu, Dashlets, Authentication
        ├── doc                 Dokumentation
        ├── library
        │   └── Training        Library-Code, Modul-Namensraum
        ├── module.info         Metadaten zum Modul
        ├── public
        │   ├── css             Eigener CSS-Code
        │   ├── img             Eigene Bilder
        │   └── js              Eigenes JavaScript
        ├── run.php             Registrierung von Hooks und mehr
        └── test
            └── php             PHP Unit-Tests

We'll go step by step through this layout during the training and fill out our project with necessary files.

## Preparing the source tree


In order to get started, we need Icinga Web 2 first. We can do it by checking out its git source tree and using it on the spot. Setting up one's webserver of choice to serve the `public` directory is one way to quickly get started. There are also simpler alternatives:

    cd /usr/local
    # if you haven't done so already:
    git clone https://git.icinga.org/icingaweb2.git
    ./icingaweb2/bin/icingacli web serve

All set to go. In order to use the installation wizard, a Token is required for security purposes. This guarantees that the web interface is not accessible by anyone else between installation and initial configuration. For packagers, this step is completely optional, as they'll be likely to roll out the installation with a CM tool, like Puppet; the configuration is located in files, so there is no need to use the wizard.

  http://localhost

## Full installation with a web server


We have so far ran Icinga Web 2 without an external web server. Even though that's good enough for sake of this training, we feel that doing a "proper" installation would be more proper course of action. We'll also make sure that PHP process starts with a blank slate, so we'll clean up its work directory:

```sh
rm -rf /tmp/FileCache_icingaweb/ /var/lib/php5/sess_*
```

These files, if placed there by root user, might cause us issues later on. Let's get the web server installed:

```
apt-get install libapache2-mod-php5
./icingaweb2/bin/icingacli setup config webserver apache \
  > /etc/apache2/conf.d/icingaweb2.conf
service apache2 restart
```

You're not mistaken here - Icinga Web 2 can generate its own configuration for Apache (2.2 but it's also compatible with 2.4). This is not limited to Apache only, as it works with nginx as well.

## The configuration structure

Unless otherwise setup, Icinga Web 2 looks for its config files in `/etc/icingaweb`. This can be overridden with environment variable ICINGAWEB_CONFIGDIR. It can be also done in web server configuration:


    SetEnv ICINGAWEB_CONFIGDIR /another/directory

## Setting up more module paths

In case you want to work with up to date version or safely switch git branches back and forth, you'll likely be reluctant to have your working copy affected by it. It is therefore recommended to use multiple module paths from the very start. This can be done in system preferences, or in the config file `/etc/icingaweb/config.ini` : 

    [global]
    module_path= "/usr/local/icingaweb-modules:/usr/local/icingaweb2/modules"


## Setting up Icinga CLI

Unless installing from repository, you'll have to set up the icingacli symlink - merely for convenience's sake :

    ln -s /usr/local/icingaweb2/bin/icingacli /usr/local/bin/

## Installing from packages


Icinga project builds weekly snapshots for various operating systems. The packages are available under [packages.icinga.org](https://packages.icinga.org/). Current build status can be seen on [build.icinga.org](https://build.icinga.org/jenkins/view/Icinga%20Web%202/) , and the current development code is available on [git.icinga.org](https://git.icinga.org/?p=icingaweb2.git) or [GitHub](https://github.com/Icinga/icingaweb2/).

For purpose of this training session, we'll be working directly on git repository. It might even make sense in production - checksumming of all files, no modified data laying around undetected, switching versions taking merely seconds - which package manager can offer you all that? Besides, this approach nicely shows how simple Icinga Web 2 really is. There was no need to change any data in git copy for installation purposes. Automake, configure - what for? The configuration is elsewhere, and that's where the runtime environment is.

# Preparing your own module

## Where do I start?

The likely important question is often, what does one need to perform with their module. In our training we shall first experiment with existing features, and eventually apply them in a practical example.


## How should i call it?

The next important problem is picking the right name for the module. Ideally it should reflect what it does. It should not be too complicated, as it will be used in PHP Namespaces, directory names and URLs.

For first baby steps, you can pick your own (Company) name. Our favorite choice for start is `training`.


## Installing and enabling a new module

    mkdir -p /usr/local/icingaweb-modules/training
    icingacli module list installed
    icingacli module enable training

All done!

# Extending the Icinga CLI

Die Icinga CLI wurde entworfen, um möglichst alles von dem was an Applikationslogik in Icinga Web 2 und dessen Modulen zur Verfügung steht auch auf der Kommandozeile bereitzustellen. Damit will das Projekt die Erstellung von Cronjobs, Plugins, nützlichen Tools und eventuell kleinen Diensten möglichst einfach gestalten.

The Icinga CLI should represent, possibly everything from the application logic that can be performed from a command line. This should provide easy way to installing cronjobs, plugins, utilities and performing other small tasks.

## Setting up Vim

We're using vim in the training, and set up a few defaults to make things more readable later:

    echo 'syntax on
    set expandtab
    set tabstop=4
    set shiftwidth=4' > /root/.vimrc

## Your own CLI commands

A structure of a cli command is:

    icingacli <module> <command> <action>

Installation of custom cli commands is quite easy. They go in directory `application/commands` in your module location, and will be invoked by their names:


    cd /usr/local/icingaweb-modules/training
    mkdir -p application/clicommands
    vim application/clicommands/HelloCommand.php

From here on, Hello (with a capital H) is a command referring to aforementioned file. The command must always come last.

Example command:

```php
<?php

namespace Icinga\Module\Training\Clicommands;

use Icinga\Cli\Command;

class HelloCommand extends Command
{
}
```

## Namespaces

* Namespaces help to define borders of your module.
* Each module receives its own namespace, named after it:

```
Icinga\Module\<Name>
```

* The starting letter MUST be uppercase.
* For CLI commands there is a dedicated namespace Clicommands

## Inheritance

All CLI commands MUST inherit the Command class in `Icinga\Cli` namespace. That brings us an entire series of examples, which we'll focus later on. It's important that your class name match the data name. In our HelloCommand.php that will be HelloCommand.

## Command actions

Each command can provide multiple actions. Each new available method that ends with Action will become a cli command action:

```php
<?php
class HelloCommand extends Command
{
    public function worldAction()
    {
        echo "Hello World!\n";
    }
}
```

## Exercise 1

Create a cli command with an Action, that will be used as follows and produce the following output:

## Bash Autocompletion

Icinga CLI provides completion for all modules, commands and actions. Installation from packages sets up everything automatically, while our manual setup requires some work:

## Bash completion

    apt-get install bash-completion
    cp etc/bash_completion.d/icingacli /etc/bash_completion.d/
    . /etc/bash_completion


When given an unclear command, this will provide on screen help.


## Inline documentation for cli commands

Commands and their actions can be easily documented with inline comments. The commentary text will be provided as help on the command line.

```php
<?php

/**
 * This is where we say hello
 *
 * The hello command allows us to be friendly to everyone
 * and his dog. That's how nice people behave!
 */
class HelloCommand extends Command
{
    /**
     * Use this to greet the world
     *
     * Greeting every single person would take some time,
     * so let's greet the whole world at once!
     */
    public function worldAction()
    {
        // ...
```

A few example commands, where you can see the help text displayed:

    icingacli training
    icingacli training hello
    icingacli help training hello
    icingacli training hello world --help

The help command can be placed at beginning, or in any other place with `--` prefix.

## Exercise 2

Provide and test documentation for a `something` action to a `say` command in the `training` module.


## Command line parameters.

We can control the command line parameters. Thanks to inheritance, we are provided with an instance of `Icinga\Cli\Params` in `$this->params` , ready to use. The object is accessible through `get()` method, which can extract the parameter values, along with option to provide a default value. Otherwise, they default to `null`.

```php
<?php

// ...

    /**
     * Say hello as someone
     *
     * Usage: icingacli training hello from --from <someone>
     */
    public function fromAction()
    {
        $from = $this->params->get('from', 'Nowhere');
        echo "Hello from $from!\n";
    }
```

### Example invocation

    icingacli training hello from --from Nürnberg
    icingacli training hello from --from "Netways Training"
    icingacli training hello from --help
    icingacli training hello from

## Standalone parameters

It is not necessary to assign values to parameters. If you wish, you can easily read in parameters in a sequence. One of easier ways is to do it with `shift()` method.

```php
<?php

// ...

/**
 * Say hello as someone
 *
 * Usage: icingacli training hello from <someone>
 */
public function fromAction()
{
    $from = $this->params->shift();
    echo "Hello from $from!\n";
}
```

### Example invocation

    icingacli training hello from Nürnberg

## Shifting is fun

The method `shift()` behaves the way programmers are already used to. First parameter from the list gets returned and discarded from it. Subsquent invocations return subsequent arguments from the list, as long as there are any left. With `unshift()`, it's possible to go back through the list.

A special case is calling `shift()` with a indicator (key) as parameter. So `shift(to)` would not only return the value to `--to` parameter, but also do it independently from its current position in the argument list. It's also possible to provide a fallback value, as before: 

```php
<?php
// ...
$person = $this->params->shift('from', 'Nobody');
```

It works the same way for standalone parameters. 
Das geht natürlich auch für Standalone-Parameter. Da wir durch den optionalen Bezeichner (key) den ersten Parameter von `shift()` schon belegt haben, jetzt aber für den zweiten (Standardwert) dennoch etwas setzen möchten, setzen wir den Bezeichner hier einfach auf null:

```php
<?php
// ...
public function fromAction()
{
    $from = $this->params->shift(null, 'Nowhere');
    echo "Hello from $from!\n";
}
```

### Beispiel-Aufruf

    icingacli training hello from Nürnberg
    icingacli training hello from
    icingacli training hello from --help

## API-Dokumentation

Die Params-Klasse im `Icinga\Cli` namespace dokumentiert noch weitere Methoden und deren Parameter. Diese sind am bequemsten in der API-Dokumentation zugänglich. Diese lässt sich mit phpDocumentor generieren, in Kürze dürfte es dazu auch einen CLI-Befehl geben.

## Aufgabe 3

Erweitere den say-Befehl, um alle folgenden Variangen zu unterstützen:

    icingacli training say hello World
    icingacli training say hello --to World
    icingacli training say hello World --from "Icinga CLI"
    icingacli training say hello World "Icinga CLI"


## Exceptions

Icinga Web 2 will sauberen PHP-Code fördern. Dazu gehört nebst anderem, dass sämtliche Warnings Fehler generieren. Zum Error-Handling werden Fehler geworfen. Wir können das einfach ausprobieren:

```php
<?php
// ...
use Icinga\Exception\ProgrammingError;
// ...
/**
 * This action will always fail
 */
public function kaputtAction()
{
    throw new ProgrammingError('No way');
}
```

### Aufruf

    icingacli training hello kaputt
    icingacli training hello kaputt --trace

## Exit-Codes

Wie wir sehen, fängt die CLI sämtliche Exceptions und gibt angenehm lesbare Fehlermeldungen nebst farbigem Hinweis auf den Fehler aus. Der Exit-Code ist in diesem Fall immer 1:

    echo $?

Damit lassen sich fehlgeschlagene Jobs zuverlässig auswerten. Nur der Exit-Code 0 steht für erfolgreiche Ausführung. Natürlich steht es jedem frei, zusätzlich weitere Exit-Codes zu benutzen. Dies erledigt man in PHP mittels `exit($code)`. Beispiel:

```php
<?php
echo "CRITICAL\n";
exit(2);
```

Alternativ stellt Icinga Web in der Command-Klasse die `fail()`-Funktion bereit. Sie ist eine Abkürzung für ein farbiges "ERROR", eine Status-Ausgabe und `exit(1)`:

```php
<?php
$this->fail('Ein Fehler ist passiert');
```

## Farben?

Wie wir eben gesehen haben, kann die Icinga CLI farbigen Output erstellen. Über die Screen-Klasse im `Icinga\Cli` Namensraum stehen nützliche Hilfsfunktionen hierzu bereit. Wir greifen in unseren Command-Klassen über `$this->screen` darauf zu. So lässt sich die Ausgabe farbig gestalten:

```php
<?php
echo $this->screen->colorize("Hello from $from!\n", 'lightblue');
```

Als optionalen dritten Parameter kann man der `colorize()`-Funktion eine Hintergrundfarbe mitgeben. Für die Darstellung der Farben werden ANSI escape codes benutzt. Erkennt Icinga CLI, dass die Ausgabe NICHT auf in ein Terminal/TTY erfolgt, werden keine Farben ausgegeben. Damit wird sichergestellt, dass z.B. beim Umleiten der Ausgabe in eine Datei keine störenden Sonderzeichen aufscheinen.

> Um das Terminal zu erkennen, wird die POSIX-Erweiterung von PHP genutzt. Ist diese nicht vorhanden, werden ebenfalls vorsichtshalber keine ANSI-Codes verwendet.

Weiter nützliche Funktionen in der Screen-Klasse sind:

* `clear()` um den Bildschirm zu löschen (wird von `--watch` benutzt)
* `underline()` um Text zu unterstreichen
* `newlines($count = 1)` um einen oder mehrere Zeilenumbrüche auszugeben
* `strlen()` um die Zeichenbreite ohne ANSI-Codes zu ermitteln
* `center($text)` um Text abhängig von der Bildschirmbreite zentriert auszugeben
* `getRows()` und `getColumns()` um wo möglich den verwendbaren Platz zu ermitteln
* `hasUtf8()` um UTF8-Unterstützung des Terminals abzufragen

Vorsicht: klappt natürlich nicht um herauszufinden, dass jemand in einem UTF8-Terminal mit einem ISO8859 Putty unterwegs ist. 

### Aufgabe

Unsere `hello`-Aktion im `say`-Befehl soll den Text in Farbe und sowohl horizontal als auch vertikal zentriert ausgeben. Wir nutzen `--watch`, um die Ausgabe abwechselnd in mindestens zwei Farben blinken zu lassen.

# Das eigene Modul im Web-Frontend

Icinga Web würde aber nicht __"Web"__ im Namen tragen, wenn seine wahren Qualitäten nicht auch dort zum Vorschein kämen. Wie wir gleich sehen werden gilt auch hier __Konvention vor Konfiguration__. Nach dem klassischen __MVC-Konzept__ gibt es natürlich Controller mit allen verfügbaren Aktionen und passende View-Skripte für die Ausgabe und Darstellung.

Wir haben bewusst auf die Trennung Library/Model verzichtet, jede zusätzliche Schicht erhöht schließlich auch die Komplexität. Man könnte auch den Library-Code in vielen Modulen als "Model" betrachten, das sollen aber die Spezialisten nach uns klären. Wir wünschen uns jedenfalls möglichst viele schicke Module, idealerweise mit viel wiederverwendbarem Code, der dann auch anderen Modulen zu Gute kommt.

## Ein erster Controller

Jede `Action` in einem `Controller` wird automatisch zu einer `Route` in unserem Web-Frontend. Das sieht in etwa wie folgt aus:

    http(s)://<host>/icingaweb/<modul>/<controller>/<action>

Wenn wir für unser Training-Modul jetzt wieder unser "Hello World" erstellen möchten, erstellen wir erst mal das Basis-Verzeichnis für unsere Controller:

    mkdir -p training/application/controllers

Anschließens legen wir unseren Controller an. Wie du schon richtig vermutest, muss dieser HelloController.php heißen und im Controllers-Namespace unseres Moduls liegen:

```php
<?php

namespace Icinga\Module\Training\Controllers;

use Icinga\Web\Controller;

class HelloController extends Controller
{
    public function worldAction()
    {
    }
}
```

Wenn wir die Url ( training/application/controllers ) jetzt aufrufen, erhalten wir eine Fehlermeldung:


    Server error: script 'hello/world.phtml' not found in path
    (/usr/local/icingaweb-modules/training/application/views/scripts/)

Praktischerweise erzählt sie uns gleich schon, was wir als nächstes machen müssen.

## Ein View-Script anlegen

Das entsprechende Basis-Verzeichnis fehlt noch. Da wir pro "action" ein View-Skript in einer dedizierten Datei anlegen, gibt es ein Verzeichnis pro "controller":

    mkdir -p training/application/views/scripts/hello
    
Das View-Skript heißt dann einfach so wie die "action", also world.phtml:

```php
<h1>Hello World!</h1>
```

Das war's auch schon, unsere neue URL ist jetzt verfügbar. Wir könnten jetzt den vollen Bereich für unser Modul nutzen und es entsprechend stylen. Wir können aber auch auf ein paar vordefinierte Elemente zurückgreifen. Zwei wichtige Klassen sind z.B. `controls` und `content` für Header-Elemente und den Seiteninhalt.

```php
<div class="controls">
<h1>Hello World!</h1>
</div>

<div class="content">
Here you go...
</div>
```

Damit erhält man automatisch gleichmäßige Abstände zu den Seitenrändern und erzielt zudem den Effekt, dass beim Scrollen nach unten die `controls` stehen bleiben, während der `content` scrollt. Das werden wir natürlich erst dann bemerken, wenn wir unser Modul mit mehr Inhalt befüllen.

## Menü-Einträge

Menü-Einträge in Icinga Web 2 können einerseits personalisiert und / oder vom Administrator vorgegeben werden (*). Unabhängig davon können sie aber von Modulen bereitgestellt werden. Hierbei handelt es sich um globale Konfiguration, die im Basis-Verzeichnis des eigenen Moduls in der `configuration.php` vorgenommen werden kann:

```php
<?php

$this->menuSection('Training')
     ->add('Hello World')
     ->setUrl('training/hello/world');
```

### Icons für Menü-Einträge

Damit unser Menüpunkt besser aussieht, verpassen wir ihm bei dieser Gelegenheit gleich noch ein Icon:

```php
<?php

$this->menuSection('Training')
     ->setIcon('thumbs-up')
     ->add('Hello World')
     ->setUrl('training/hello/world');
```

Um herauszufinden, welche Icons zur Verfügung stehen, aktivieren wir unter `System` / `Module` das `doc`-Modul. Anschließend finden wir die Icon-Liste unter `Dokumentation` / `Developer - Style`. Es handelt sich hierbei um Icons welche in eine Schriftart eingebettet wurden. Das hat den großen Vorteil, dass sehr viel weniger Requests über die Leitung müssen - die Icons sind einfach "immer da".

Alternativ lassen sich auf Wunsch aber immer noch klassische Icons (.png etc) benutzen. Das ist vor allem dann nützlich, wenn man für sein Modul ein spezielles Icon (z.B. ein Firmenlogo) nutzen möchte, welches sich schlecht in den offiziellen Icinga Icon-Font integrieren lässt:

```php
<?php

$this->menuSection('Training')->setIcon('img/icons/success.png');
```

## Bilder hinzufügen

Wenn man in seinem Modul eigene Bilder nutzen möchte, stellt man diese einfach unter `public/img` bereit:

    mkdir -p public/img
    wget https://www.icinga.org/wp-content/uploads/2014/06/tgelf.jpg
    mv tgelf.jpg public/img/

Unsere Bilder sind sofort via Web erreichbar, das URL-Muster ist wie folgt:

    http(s)://<icingaweb>/img/<module>/<bild>

Für unseren konkreten fall also http://localhost/img/training/tgelf.jpg. Das lässt sich so auch wunderbar gleich in unserem View-Skript nutzen. Anstatt einen img-Tag anzulegen (was natürlich möglich wäre) nutzen wir einen der vielen praktischen View-Helper:

```php
...
<div class="content">
<?= $this->img('img/training/tgelf.jpg', array('title' => 'Thomas Gelf')) ?> Here you go...
</div>
```

## Aufgabe

Erstelle die URLs `training/hello/thomas` und `training/say/hello` und füge jeweils einen zusätzlichen Menüpunkt hinzu. Suche zudem für unser Training-Modul ein schöneres Icon aus dem Internet und richte es entsprechend ein.

## Dashboards

Bevor wir uns um ernsthafte Themen kümmern wollen wir unsere nützliche URL natürlich noch als Default-Dashboard bereitstellen. Auch das lässt sich in der `configuration.php` erledigen:

```php
<?php
$this->dashboard('Training')->add('Hello', 'training/hello/world');
```

# Wir brauchen Daten!

Nachdem unsere Web-Routen jetzt so wunderbar funktionieren, wollen wir natürlich etwas Sinnvolles damit anstellen. Eine Anwendung kann noch so schön sein, ohne nützliche Inhalte wird sie schnell langweilig. In einer MVC-Umgebung besorgen sich für gewöhnlich die `Controller` mit Hilfe der `Models` ihre Daten und befüttern damit die `View`.

## Unsere View mit Daten befüllen

Der Controller stellt in `$this->view` einen Zugriff auf unsere View bereit. Auf diesem Wege lässt sie sich ganz bequem betanken:

```php
<?php

public function worldAction()
{
    $this->view->application = 'Icinga Web 2';
    $this->view->moreData = array(
        'Work'   => 'done',
        'Result' => 'fantastic'
    );
}
```

Wir erweitern jetzt unser View-Skript und stellen die übermittelten Daten entsprechend dar:

```php
<h3>Some data...</h3>

This example is provided by <a href="http://www.netways.de">Netways</a> 
and based on <?= $this->application ?>.

<table>
<?php foreach ($this->moreData as $key => $val): ?>
    <tr><th><?= $key ?></th><td><?= $val ?></td></tr>
<?php endforeach ?>
</table> 
```

## Aufgabe

Unter `training/list/files` soll der Inhalt unseres Modul-Verzeichnisses in Tabellen-Form aufgelistet werden.
* Hinweis: mit `$this->Module()->getBaseDir()` ermitteln wir unser Modul-Verzeichnis
* Mehr zum öffnen von Verzeichnissen unter [http://de.php.net/opendir](http://de.php.net/opendir)

# Aber bitte mit Stil!

Das hat jetzt zwar nicht direkt mit unserem Thema zu tun, aber eins fällt auf: unsere Tabelle ist nicht gerade besonders hübsch. Zum Glück können wir in unser Modul ganz bequem auch CSS packen. Wir erstellen dazu ein passendes Verzeichnis, der Name dürfte naheliegend sein:

    mkdir public/css

Unsere CSS-Anweisungen legen wir anschließend dort in der Datei `module.less` ab. Less ist eine CSS-Erweiterung um allerhand Funktionen, mehr dazu findet sich unter [...](). Herkömmliches CSS ist hier aber auf jeden Fall gültig. Das Schöne an Icinga Web ist nun, dass ich mir keine Gedanken darüber machen muss, ob mein CSS andere Module oder Icinga Web selbst beeinflusst: das ist nicht der Fall.

So können wir problemlos folgendes definieren, ohne fremde Tabellen "kaputt" zu machen:

    table {
        width: 100%;
    }

    th {
        width: 20%;
        text-align: right;
        line-height: 2em;
        padding-right: 2em;
    }

Wenn wir in den Entwickler-Tools unseres Browsers die Requests beobachten sehen wir, dass Icinga Web als einzige CSS-Datei css/icings.min.css lädt. Wir können auch css/icinga.css laden um uns bequem anschauen zu können, was Icinga Web aus unserem CSS-Code gemacht hat:

    .icinga-module.module-training table {
      width: 100%;
    }
    .icinga-module.module-training th {
      width: 20%;
      text-align: right;
      line-height: 2em;
      padding-right: 2em;
    }

Wie wir sehen wird durch entsprechende Präfixe sichergestellt, dass unser CSS immer nur in jenen Containern gilt, in denen unser Modul seine Inhalte darstellt.

## Nützliche CSS-Klassen

Icinga Web 2 stellt eine Reihe von CSS-Klassen bereit, die uns die Arbeit einfacher machen. So ist `common-table` nützlich für die üblichen Listen in Tabellen, `name-value-table` Für Name/Wert-Paare bei denen links der Bezeichner als th und rechts der entsprechende Wert in einem td dargestellt wird. Hilfreich ist auch `table-row-selectable` - damit verändert sich das Verhalten der Tabelle. Die ganze Zeile wird hervorgehoben, wenn man mit der Maus darüber fährt. Wenn man irgendwo klickt, kommt der erste Link der Zeile zum Zug. In Kombination mit `common-table` sieht das Ganze ohne zusätzliche Arbeit dann auch schon gleich gut aus.

# Echte Daten aufgeräumt

Wie wir vorhin gesehen haben, wird so ein Modul erst mit echten Daten so richtig interessant. Was wir allerdings falsch gemacht haben ist, dass unser Controller die Daten selbst besorgt. Das ist unschön und würde uns spätestens wenn wir diese Daten auch auf der CLI nutzen möchten Probleme bereiten.

## Unsere eigene Bibliothek

Wir erstellen für unsere Bibliothek wieder ein neues Verzeichnis in unserem Module und folgen dabei dem Schema `library/<Modulname>`. In unserem Fall also:

    mkdir -p library/Training

Für unser Modul nutzen wir wie schon gelernt den Namensraum `Icinga\Module\<Modulname>`. Alle darunter befindlichen Namespaces sucht Icinga Web 2 automatisch im eben erstellten Verzeichnis. Ausnahmen sind die vorhin gesehen, wie z.B. `Clicommands` oder `Controllers`. 

Eine Biblithek, welche die in der Übung umgesetzte Aufgabe erledigt, könnte in `File.php` liegen und wie folgt aussehen:

```php
<?php

namespace Icinga\Module\Training;

use DirectoryIterator;

class Directory
{
    public static function listFiles($path)
    {
        $result = array();

        foreach (new DirectoryIterator($path) as $file) {
            if ($file->isDot()) continue;

            $result[] = (object) array(
                'name' => $file->getFilename(),
                'path' => $file->getPath(),
                'size' => $file->getSize(),
                'type' => $file->getType()
            );
        }
        return $result;
    }
}
```

Unser Controller kann die Daten jetzt ganz bequem über unsere kleine Library abholen:

```php
<?php

// ...
use Icinga\Module\Training\File;

class FileController extends Controller
{
    public function listAction()
    {
        $this->view->files = File::list($this->Module()->getBaseDir());
    }
}
```

## Aufgabe

Setze diese oder eine vergleichbare Library in dein Modul. Stelle ein View-Skript bereit, welches passend dazu die einzelnen Files auflisten kann. Wichtig dabei: benutze `$this->escape()` im View-Skript, um Daten deren Herkunft unsicher ist (z.B. Dateinamen) entsprechend zu escapen.

# Parameter-Handling

Bisher haben wir an unsere URLs noch keine Parameter mitgegeben. Auch das ist aber ganz einfach. Wie auf der Kommandozeile steht uns in Icinga Web ein simpler Zugriff auf Params zur Verfügung. Der Zugriff darauf erfolgt wie gewohnt:

```php
<?php
$file = $this->params->get('file');
```

Auch `shift()` und Konsorten sind hier natürlich wieder verfügbar.

## Aufgabe

Unter `training/file/show?file=<filename>` sollen nach Belieben zusätzliche Infos zur gewünschten Datei angezeigt werden. Fleißige zeigen Eigentümer, Berechtigungen, letzte Änderung und Mime-Type an - es reicht aber auch völlig einfach nur erneut Dateiname und Größe "in schön" darzustellen.

## Weiterführende Links

In unserer Dateiliste wollen wir jetzt von jeder Datei zum entsprechenden Detailbereich verlinken. Um keine Probleme mit dem Escaping von Parametern zu bekommen, nutzen wir einen neuen Helper, `qlink`:

```php
<td><?= $this->qlink(
    $file->name,
    'training/file/show',
    array('file' => $file->name)
) ?></td>
```

Der erste Parameter ist hierbei der anzuzeigende Text, der zweite der zu erstellende Link und an dritter Stelle optionale Paramteter für diesen Link. Als vierten Parameter könnte man noch ein Array mit beliebigen weiteren HTML-Attributen mitgeben.

Wenn wir jetzt in unserer Liste eine Datei anklicken, landen wir bei den entsprechenden Details dazu. Doch das geht auch bequemer. Probiere einfach mal, `data-base-target="_next"` in das content-div zu setzen:

    <div class="content" data-base-target="_next">

Damit steuern wir erstmals ohne großen Aufwand das mehrspaltige Layout von Icinga Web!

# URL-Handling

Wer beobachtet hat wie der Browser sich verhält, der hat vielleicht bemerkt, dass hier nicht bei jedem Klick die Seite neu geladen wird. Icinga Web 2 fängt sämtliche Requests ab und versendet sie eigenständig per XHR-Request. Serverseitig wird dies erkannt, und als Antwort dann lediglich das jeweilige HTML-Schnipsel versendet. Das entspricht meist lediglich dem vom entsprechenden View-Script erstellten Output.

Trotzdem bleibt jeder Link ein Link und lässt sich z.B. in einem neuen Tab öffnen. Hier wiederum wird erkannt, dass es sich um keinen XHR-Request handelt, das vollständige Layout wird ausgeliefert.

Für gewöhnlich landen Links immer im selben Container, man kann das Verhalten mit `data-base-target` aber beeinflussen. Das am nächsten am angeklickten Element liegende Attribut gewinnt dabei. Will man `_next` für einen Teilbereich der Seite wieder aufheben, setzt man dort einfach `data-base-target="_self"`.

# Daten-Handling leichtgemacht

Icinga Web bietet noch einiges an netten Hilfmitteln. Eines wollen wir noch begutachten, die sogenannten DataSources. Wir binden die ArrayDatasource ein und erweitern unseren Library-Code um eine weitere Funktion:

```php
<?php

use Icinga\Data\DataArray\ArrayDatasource;

// ...

    public static function selectFiles($path)
    {
        $ds = new ArrayDatasource(self::listFiles($path));
        return $ds->select();
    }
```

Anschließend ändern wir auch unseren Controller ganz leicht ab:


```php
<?php
$query = Directory::selectFiles(
    $this->Module()->getBaseDir()
)->order('type')->order('name');

$this->view->files = $query->fetchAll();
```

## Aufgabe 1
Baue die Liste so um, dass man sie per Mausklick auf- oder absteigend sortieren kann.

## Zusatzaufgabe

```php
<?php

$editor = Widget::create('filterEditor')->handleRequest($this->getRequest());
$query->applyFilter($editor->getFilter());
```

## Autorefresh

Als Monitoring-Oberfläche ist es selbstverständlich, dass Icinga Web eine zuverlässig und stabile Autorefresh-Funktion mitliefert. Diese lässt sich bequem aus den Controllern steuern:

```php
<?php

$this->setAutorefreshInterval(10);
```

## Aufgabe 2

Unsere Datei-Liste soll automatisch aktualisiert werden, die Detail-Infos ebenfalls. Zeige die Änderungszeit einer Datei an (`$file->getMtime()`) und benutze den Helper `timeSince` um die Zeit darzustellen. Ändere eine Datei auf der Festplatte und beobachte, was passiert. Wie kann man das erklären?

# Konfiguration

Wer ein Modul entwickelt möchte dieses vermutlich auch konfigurieren können. Konfiguration für ein Module legt man unter `/etc/icingaweb/modules/<modulename>` ab. Was sich dort in einer `config.ini` findet ist im Controller wie folgt zugänglich:

```php
<?php
$config = $this->Config();

/*
[abschnitt]
eintrag = "wert"
*/
echo $config->get('abschnitt', 'eintrag');

// Ergibt "standardwert", da "keineintrag" nicht existiert:
echo $config->get('abschnitt', 'keineintrag', 'standardwert');

// Liest aus der special.ini statt aus der config.ini:
$config = $this->Config('special');
```

## Aufgabe
Der Basis-Pfad für den List-Controller unseres Training-Moduls soll konfigurierbar sein. Ist kein Pfad konfiguriert, benutzen wir weiterhin unser Modulverzeichnis.

# Übersetzungen
Für eine detaillierte Beschreibung der Übersetzungsmöglichkeiten öffen wir die Dokumentation zum `translation` Modul. Hier kurz die einzelnen Schritte:

```php
<h1><?= $this->translate('My files') ?></h1>
```

    apt-get install gettext poedit
    icingacli module enable translation
    icingacli translation refresh module training de_DE
    # Übersetzen mit Poedit
    icingacli translation compile module training de_DE


# Icinga Web Logik in Drittsoftware nutzen?

Wir wollen mit Icinga Web 2 nicht nur das Einbinden von Drittsoftware möglichst einfach gestalten. Wir wollen auch, dass es anderen einfach fällt, Icinga Web Logik in deren Software zu nutzen.

Dazu reicht im Grunde folgender Aufruf in einer beliebigen PHP-Datei:

```php
<?php

require_once 'Icinga/Application/EmbeddedWeb.php';
Icinga\Application\EmbeddedWeb::start();
```

Fertig! Keine Authentifizierung, kein Bootstrapping der vollen Web-Oberfläche. Aber alles was an Library-Code vorhanden ist, kann genutzt werden.

## Aufgabe
Erstelle eine zusätzliche PHP-Datei, welche Icinga Web 2 einbettet. Benutze anschließend die Bibliothek zum Verzeichnis-Handling aus deinem Trainingsmodul.

# Freies Lab

Du bist wirklich hier angekommen? An einem einzigen Tag? Respekt. Der Referent hat jetzt garantiert eine spannende letzte Übungsaufgabe parat, um den Tag mit einem praxisnahen Beispiel und vielen neuen Tricks ausklingen zu lassen.

Viel Freude mit Icinga Web 2!!!

