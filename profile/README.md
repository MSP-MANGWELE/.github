# DOCUMENTATION TECHNIQUE GENERALE (DTIC)
Ce document aborde plusieurs aspects pour le developpement des differentes applications réalisés par la DTIC, ces partenaires et ces consultants.

## Table des matières
- [Normes et conventions Laravel](#normes-et-conventions-laravel)
- [Normes et Conventions Flutter](#normes-et-conventions-flutter)
- [Documentation de Projet pour les Applications Mobiles Mangwele et Mavimpi](#documentation-de-projet-pour-les-applications-mobiles-mangwele-et-mavimpi)

---------------------------------------------------------------------------------------------------

![Laravel Logo](https://upload.wikimedia.org/wikipedia/commons/9/9a/Laravel.svg)

# **Normes et conventions Laravel**

Cette documentation technique vise à fournir des lignes directrices et des normes pour le développement avec le framework Laravel au sein de la Direction des Technologies de l'Information du Ministère de la Santé et de la Population. Elle couvre une variété de bonnes pratiques et conventions à suivre pour assurer la qualité, la maintenabilité et la cohérence du code. Les thèmes abordés incluent le principe de responsabilité unique, l'utilisation de modèles et de contrôleurs, la validation, l'organisation de la logique métier, et l'importance de ne pas se répéter. De plus, des recommandations sont faites concernant l'utilisation d'Eloquent plutôt que de requêtes SQL brutes, l'affectation en masse, et la gestion des requêtes dans les modèles Blade. En suivant ces conventions, les développeurs peuvent créer des applications Laravel plus robustes, efficaces et faciles à maintenir.

## Table des matières

- [Principe de responsabilité unique](#principe-de-responsabilité-unique)

- [Gros Modèles, contrôleurs maigres](#gros-modèles-contrôleurs-maigres)

- [Validation](#validation)

- [La logique métier doit être dans une classe de service](#la-logique-métier-doit-être-dans-une-classe-de-service)

- [Ne te répète pas (DRY)](#ne-te-répète-pas-dry)

- [Préférez utiliser Eloquent à l’utilisation de Query Builder et de requêtes SQL brutes. Préférez les collections aux tableaux](#préférez-utiliser-eloquent-à-lutilisation-de-query-builder-et-de-requêtes-sql-brutes-préférez-les-collections-aux-tableaux)

- [Affectation en masse](#affectation-en-masse)

- [N'exécutez pas de requêtes dans les modèles de blade et utilisez un chargement rapide (N + 1 problème)](#nexécutez-pas-de-requêtes-dans-les-modèles-blade-et-utilisez-un-chargement-rapide-eager-loading-problème-n--1)

- [Commentez votre code, mais préférez une méthode descriptive et les noms de variables aux commentaires](#commentez-votre-code-mais-préférez-une-méthode-descriptive-et-des-noms-de-variables-aux-commentaires)

- [Ne mettez pas JS et CSS dans les templates Blade et ne mettez pas de HTML dans les classes PHP](#ne-mettez-pas-js-et-css-dans-les-templates-blade-et-ne-mettez-pas-de-html-dans-les-classes-php)

- [Utilisez des fichiers de configuration et de langue, des constantes au lieu du texte dans le code](#utilisez-des-fichiers-de-configuration-et-de-langue-des-constantes-au-lieu-du-texte-dans-le-code)

- [Utiliser les outils standard de Laravel acceptés par la communauté](#utiliser-les-outils-standard-de-laravel-acceptés-par-la-communauté)

- [Suivre les conventions de nommage de Laravel](#suivre-les-conventions-de-nommage-de-laravel)

- [Utilisez une syntaxe plus courte et plus lisible dans la mesure du possible](#utilisez-une-syntaxe-plus-courte-et-plus-lisible-dans-la-mesure-du-possible)

- [Utilisez un conteneur IoC ou des façades au lieu de la nouvelle classe](#utilisez-un-conteneur-ioc-ou-des-façades-au-lieu-de-la-nouvelle-classe)

- [Ne pas obtenir directement les données du fichier `.env` directement](#ne-pas-obtenir-directement-les-données-du-fichier-env)

- [Stocker les dates au format standard. Utiliser des accesseurs et des mutateurs pour modifier le format de date](#stocker-les-dates-au-format-standard-Utiliser-des-accesseurs-et-des-mutateurs-pour-modifier-le-format-de-date)

- [D'autres bonnes pratiques](#dautres-bonnes-pratiques)


### **Principe de responsabilité unique**

Une classe et une méthode ne devraient avoir qu'une seule responsabilité.

Mal:

```php
public function getFullNameAttribute(): string
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Bien:

```php
public function getFullNameAttribute(): string
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient(): bool
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong(): string
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort(): string
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 Retour au contenu](#contents)

### **Gros modèles, contrôleurs maigres**

Placez toute la logique liée à la base de données dans les modèles Eloquent ou dans les classes du Repository si vous utilisez le générateur de requêtes ou des requêtes SQL brutes.

Mal:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Bien:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 Retour au contenu](#contents)

### **Validation**

Déplacez la validation des contrôleurs vers les classes Request.

Mal:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ...
}
```

Bien:

```php
public function store(PostRequest $request)
{
    ...
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 Retour au contenu](#contents)

### **La logique métier doit être dans une classe de service**

Un contrôleur ne doit avoir qu'une seule responsabilité. Par conséquent, déplacez la logique métier des contrôleurs vers les classes de service.

Mal:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ...
}
```

Bien:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ...
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 Retour au contenu](#contents)

### **Ne te répète pas (DRY)**

Réutilisez le code quand vous le pouvez. SRP vous aide à éviter les doubles emplois. Réutilisez également les modèles Blade, utilisez les Eloquent scopes, etc.

Mal:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Bien:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 Retour au contenu](#contents)

### **Préférez utiliser Eloquent à l’utilisation de Query Builder et de requêtes SQL brutes. Préférez les collections aux tableaux**

Eloquent vous permet d’écrire du code lisible et maintenable. Eloquent dispose également d'excellents outils intégrés tels que les suppressions, les événements, les scopes, etc.

Mal:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Bien:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 Retour au contenu](#contents)

### **Affectation en masse**

Mal:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;

// Add category to article
$article->category_id = $category->id;
$article->save();
```

Bien:

```php
$category->article()->create($request->validated());
```

[🔝 Retour au contenu](#contents)

### **N'exécutez pas de requêtes dans les modèles Blade et utilisez un chargement rapide (eager loading) (problème N + 1)**

Mal (Pour 100 utilisateurs, 101 requêtes DB seront exécutées):

```blade
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Bien (pour 100 utilisateurs, 2 requêtes de base de données seront exécutées):

```php
$users = User::with('profile')->get();

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 Retour au contenu](#contents)

### **Commentez votre code, mais préférez une méthode descriptive et des noms de variables aux commentaires**

Mal:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Meilleure:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Bien:

```php
if ($this->hasJoins())
```

[🔝 Retour au contenu](#contents)

### **Ne mettez pas JS et CSS dans les templates Blade et ne mettez pas de HTML dans les classes PHP**

Mal:

```javascript
let article = `{{ json_encode($article) }}`;
```

Meilleure:

```php
<input id="article" type="hidden" value='@json($article)'>

Or

<button class="js-fav-article" data-article='@json($article)'>{{ $article->name }}<button>
```

Dans un fichier Javascript:

```javascript
let article = $('#article').val();
```

Le meilleur moyen consiste à utiliser un package PHP vers JS spécialisé pour transférer les données.

[🔝 Retour au contenu](#contents)

### **Utilisez des fichiers de configuration et de langue, des constantes au lieu du texte dans le code**

Mal:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Bien:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 Retour au contenu](#contents)

### **Utiliser les outils standard de Laravel acceptés par la communauté**

Préférez utiliser les fonctionnalités intégrées de Laravel et les packages de communauté au lieu d'utiliser des packages et des outils tiers. Tout développeur qui travaillera avec votre application à l'avenir devra apprendre de nouveaux outils. En outre, les chances d'obtenir de l'aide de la communauté Laravel sont considérablement réduites lorsque vous utilisez un package ou un outil tiers. Ne faites pas payer votre client pour cela.

Tâche | Outils standard | Outils tiers
------------ | ------------- | -------------
Autorisation | Policies | Entrust, Sentinel et d'autres packages
Compiler des assets | Vite | Grunt, Gulp, packages tiers
Environnement de développement | Laravel Sail, Homestead | Docker
Déploiement | Serveur Linux | Deployer et d'autre solutions
Tests unitaires | PHPUnit, Mockery | Phpspec, Pest
Test du navigateur | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Travailler avec des données | Laravel collections | Arrays
Validation du formulaire | Request classes | 3rd party packages, validation dans le contrôleur
Authentification | Built-in | 3rd party packages, votre propre solution
API D'authentification | Laravel Passport, Laravel Sanctum | 3rd party JWT et OAuth packages
Création d'API | Built-in | Dingo API and similar packages
Travailler avec une structure de base de données | Migrations | Travailler directement avec la structure de la base de données
Localisation | Built-in | 3rd party packages
Interfaces utilisateur en temps réel | Laravel Echo, Pusher | Packages tiers et utilisation directe de WebSockets
Générer des données de test | Seeder classes, Model Factories, Faker | Création manuelle de données de test
Planification des tâches | Laravel Task Scheduler | Scripts et packages tiers
DB | MySQL, PostgreSQL, SQLite | MongoDB

[🔝 Retour au contenu](#contents)

### **Suivre les conventions de nommage de Laravel**

Suivre [Normes PSR](https://www.php-fig.org/psr/psr-12/).

Suivez également les conventions de nommage acceptées par la communauté Laravel:

Quoi | Comment | Bien | Mal
------------ | ------------- | ------------- | -------------
Controller | singulier | ArticleController | ~~ArticlesController~~
Route | pluriel | articles/1 | ~~article/1~~
Route nommée | snake_case avec notation par points | users.show_active | ~~users.show-active, show-active-users~~
Model | singulier | User | ~~Users~~
Relations hasOne or belongsTo | singulier | articleComment | ~~articleComments, article_comment~~
Toutes les autres relations | pluriel | articleComments | ~~articleComment, article_comments~~
Table | plurielle | article_comments | ~~article_comment, articleComments~~
Table pivot | noms des modèles au singulier dans l'ordre alphabétique | article_user | ~~user_article, articles_users~~
Colonne de table | snake_case sans nom de modèle | meta_title | ~~MetaTitle; article_meta_title~~
Attribut du Model | snake_case | $model->created_at | ~~$model->createdAt~~
Foreign key | Nom du modèle au singulier avec _id comme suffix | article_id | ~~ArticleId, id_article, articles_id~~
Primary key | - | id | ~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Méthode | camelCase | getAll | ~~get_all~~
Méthodes dans le controlleur de ressource | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Méthode dans une classe de test | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variable | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptif, pluriel | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptif, singulier | $activeUser = User::active()->first() | ~~$users, $obj~~
Index de fichier de config et de langage | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
Vue | kebab-case | show-filtered.blade.php | ~~showFiltered.blade.php, show_filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjectif ou nom | AuthenticationInterface | ~~Authenticatable, IAuthentication~~
Trait | adjectif | Notifiable | ~~NotificationTrait~~
Trait [(PSR)](https://www.php-fig.org/bylaws/psr-naming-conventions/) | adjective | NotifiableTrait | ~~Notification~~
Enum | singular | UserType | ~~UserTypes~~, ~~UserTypeEnum~~
FormRequest | singular | UpdateUserRequest | ~~UpdateUserFormRequest~~, ~~UserFormRequest~~, ~~UserRequest~~
Seeder | singular | UserSeeder | ~~UsersSeeder~~

[🔝 Retour au contenu](#contents)

### **Utilisez une syntaxe plus courte et plus lisible dans la mesure du possible**

Mal:

```php
$request->session()->get('cart');
$request->input('name');
```

Bien:

```php
session('cart');
$request->name;
```

Plus d'exemples:

Syntaxe commune | Syntaxe plus courte et plus lisible
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝 Retour au contenu](#contents)

### **Utilisez un conteneur IoC ou des façades au lieu de la nouvelle classe**

La nouvelle syntaxe de classe crée un couplage étroit entre les classes et complique les tests. Utilisez plutôt le conteneur IoC ou les façades.

Mal:

```php
$user = new User;
$user->create($request->validated());
```

Bien:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

...

$this->user->create($request->validated());
```

[🔝 Retour au contenu](#contents)

### **Ne pas obtenir directement les données du fichier `.env`**

Passez les données aux fichiers de configuration à la place, puis utilisez la fonction d'assistance `config ()` pour utiliser les données dans une application.

Mal:

```php
$apiKey = env('API_KEY');
```

Bien:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[🔝 Retour au contenu](#contents)

### **Stocker les dates au format standard. Utiliser des accesseurs et des mutateurs pour modifier le format de date**

Mal:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Bien:

```php
// Model
protected $casts = [
    'ordered_at' => 'datetime',
];

public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[🔝 Retour au contenu](#contents)

### **D'autres bonnes pratiques**

Ne mettez jamais aucune logique dans les fichiers de routes.

Minimisez l'utilisation de PHP vanilla dans les modèles de blade.

[🔝 Retour au contenu](#contents)



---------------------------------------------------------------------------------------------------------------------



![Flutter Logo](https://upload.wikimedia.org/wikipedia/commons/1/17/Google-flutter-logo.png)

# **Normes et Conventions Flutter**

Cette documentation fournit des lignes directrices et des normes pour le développement avec le framework Flutter. Ces conventions visent à assurer la qualité, la maintenabilité et la cohérence du code. Elles couvrent une variété de bonnes pratiques, allant de la structure du projet à la gestion de l'état et l'utilisation de widgets.

## Table des matières

- [Structure du projet](#structure-du-projet)
- [Nom des fichiers et des classes](#nom-des-fichiers-et-des-classes)
- [Formatage du code](#formatage-du-code)
- [Utilisation des Widgets](#utilisation-des-widgets)
- [Gestion de l'état](#gestion-de-létat)
- [Navigation](#navigation)
- [Bonnes pratiques pour les performances](#bonnes-pratiques-pour-les-performances)
- [Internationalisation](#internationalisation)
- [Tests](#tests)
- [Documentation et commentaires](#documentation-et-commentaires)
- [Sécurité](#sécurité)

### Structure du projet

Adoptez une structure de projet standard pour organiser votre code de manière claire et compréhensible.

```
lib/
|-- src/
|   |-- models/
|   |-- views/
|   |-- controllers/
|   |-- services/
|   |-- utils/
|-- main.dart
```

- **models/**: Contient les classes de modèles de données.
- **views/**: Contient les widgets et les écrans de l'interface utilisateur.
- **controllers/**: Contient la logique métier et les contrôleurs.
- **services/**: Contient les services pour la communication avec des APIs ou des bases de données.
- **utils/**: Contient les utilitaires et les fonctions d'aide.

### Nom des fichiers et des classes

Suivez des conventions de nommage claires pour améliorer la lisibilité et la maintenabilité du code.

- Les fichiers doivent être nommés en snake_case (par exemple, `user_profile.dart`).
- Les classes doivent être nommées en PascalCase (par exemple, `UserProfile`).

### Formatage du code

Utilisez `flutter format` pour formater automatiquement votre code conformément aux conventions de style Dart.

- Limitez les lignes à 80 caractères.
- Utilisez des accolades pour toutes les structures conditionnelles et boucles, même si elles contiennent une seule instruction.
- Indentez avec deux espaces, pas de tabulations.

### Utilisation des Widgets

Respectez les bonnes pratiques suivantes pour l'utilisation des widgets:

- Utilisez des widgets StatelessWidget autant que possible pour des raisons de performance.
- Utilisez des widgets StatefulWidget uniquement lorsque l'état local est nécessaire.
- Séparez les widgets complexes en plusieurs widgets plus petits et réutilisables.

### Gestion de l'état

Choisissez une solution de gestion de l'état adaptée à la taille et à la complexité de votre application. Dans le cadre des applications Mangwele et Mavimpi, nous utiliserons Getx pour la gestion de l'état en raison de ses nombreuses fonctionnalités. Cette classification ne vise pas à comparer les fonctionnalités des différentes solutions, mais à fournir des recommandations en fonction de la complexité de l'application.

- Pour des applications simples, utilisez `setState`.
- Pour des applications de taille moyenne, utilisez `Provider`.
- Pour des applications complexes, envisagez des solutions comme `Getx` ou `Bloc`.

### Navigation

Utilisez le package `flutter_navigation` pour une navigation claire et concise.

- Définissez toutes les routes dans un fichier central `routes.dart`.
- Utilisez `Navigator.pushNamed` pour la navigation basée sur des noms de routes.

Si vous utilisez Getx, profitez de son système de navigation intégré pour simplifier et optimiser la gestion des routes et des transitions entre les pages.

### Bonnes pratiques pour les performances

- Minimisez l'utilisation de widgets redondants.
- Utilisez des `const` constructors lorsque possible.
- Privilégiez les `ListView.builder` au lieu des `ListView` pour les longues listes.
- Utilisez les outils de profilage de Flutter pour identifier et résoudre les problèmes de performance.

### Internationalisation

Assurez-vous que votre application est accessible à un public mondial en implémentant l'internationalisation. Pour les applications Mangwele et Mavimpi, nous utiliserons Getx pour gérer l'internationalisation, grâce à ses nombreuses fonctionnalités pratiques. Cette recommandation n'a pas pour but de comparer les fonctionnalités des différentes solutions d'internationalisation, mais de fournir une méthode adaptée à la complexité de l'application.

- Utilisez le package `flutter_localizations` pour supporter plusieurs langues.
- Définissez toutes les chaînes de caractères dans des fichiers de localisation centralisés, par exemple `en.json` et `fr.json`.
- Utilisez `Localizations.of(context)` pour accéder aux chaînes localisées dans votre application.

Lorsque Getx est utilisé, privilégiez son système intégré d'internationalisation pour une gestion efficace et simplifiée des différentes langues.

```dart
import 'package:get/get.dart';

class Messages extends Translations {
  @override
  Map<String, Map<String, String>> get keys => {
    'en_US': {
      'hello': 'Hello',
    },
    'fr_FR': {
      'hello': 'Bonjour',
    },
  };
}
```

Dans ce cas, vous pouvez configurer Getx pour gérer les traductions de manière centralisée et dynamique, rendant votre application plus adaptable aux besoins multilingues.

### Tests

- Écrivez des tests unitaires pour la logique métier.
- Écrivez des tests d'intégration pour les interactions entre les composants.
- Utilisez `flutter_test` pour les tests de widgets.
- Utilisez `mockito` ou `bloc_test` pour les tests de mocks et de blocs respectivement.

### Documentation et commentaires

- Documentez toutes les classes et méthodes publiques avec des commentaires de documentation.
- Utilisez des commentaires pour expliquer les parties complexes du code, mais préférez un code clair et auto-documenté.
- Utilisez des commentaires TODO pour indiquer les parties du code nécessitant des améliorations ou des fonctionnalités futures.

### Sécurité

- Ne stockez jamais d'informations sensibles en clair dans le code.
- Utilisez des packages comme `flutter_secure_storage` pour le stockage sécurisé des données sensibles.
- Validez et vérifiez toutes les entrées utilisateur pour éviter les failles de sécurité telles que les injections SQL et les XSS.

En suivant ces normes et conventions, les développeurs peuvent créer des applications Flutter plus robustes, maintenables et cohérentes.




--------------------------------------------------------------------------------------------------------------------------------------------------
# **Documentation de Projet pour les Applications Mobiles Mangwele et Mavimpi**

## Introduction

Ce document explique la réalisation et le fonctionnement des applications Mangwele et Mavimpi, toutes deux destinées à des usages communautaires en République du Congo.

- **Mavimpi** : Une application permettant de récolter des informations sur les ménages à travers tout le territoire, utilisée par les agents communautaires pour diverses fins statistiques et administratives.
- **Mangwele** : Une application de collecte de données de vaccination et de rappel vaccinal par SMS, utilisée pour améliorer la couverture vaccinale et le suivi des vaccinations en République du Congo.

## Architecture de l'Application

Les deux applications sont construites suivant le modèle MVVM (Model-View-ViewModel), une architecture visant à séparer la logique des interfaces utilisateurs de la logique métier. Cette approche garantit une structure bien définie et facile à maintenir.

### Structure de l'Application

#### 1. Components

- **Composants** : Regroupe tous les widgets UI réutilisables pour les différents écrans de l'application.

#### 2. Composables

- **Fonctions d'appel API** : Définit la logique de chaque appel API, incluant l'URL appelée, la méthode HTTP et les données envoyées, si elles existent.
- **Fonction de redirection** : Gère les réponses reçues des API pour rediriger les utilisateurs en fonction des réponses.

#### 3. Storage

- **Classe de stockage** : Gère le stockage local d'informations comme le token et d'autres données utilisateur nécessaires pour certaines opérations.

#### 4. Models

- **Modèles** : Contient toutes les classes modèles et référentiels utilisés pour le traitement des données, l’hydratation des objets, et l'affichage des informations.

#### 5. Modules

Chaque module représente une partie distincte de l'application et contient :
- **Vue (View)** : Code UI.
- **Contrôleur (Controller)** : Logique de l'application.
- **Service** : Appels API.

#### 6. Resources

- **Fichiers de traduction** : Pour l'internationalisation.
- **Fichier de couleurs** : Définition des couleurs de l'application.
- **Autres fichiers de ressources** : Comme les polices et les icônes.

#### 7. Utils

- **Fonctions utilitaires** : Contient des fonctions réutilisables dans toute l'application, telles que des fonctions de conversion de date et d'heure.

#### 8. Routes

- **Gestion des routes** : Définit toutes les routes pour la navigation dans l'application.

## Technologie, Framework et Bibliothèque Utilisés

- **Langue de programmation** : Dart
- **Framework** : Flutter
- **Bibliothèque** :
  - `getx`
  - `flutter_secure_storage`
  - `http`
  - `flutter_screen_utils`
  - `sqflite`
  - `path`

Pour plus d'informations sur ces bibliothèques, consultez [pub.dev](https://www.pub.dev).

## Conclusion

Cette documentation fournit un aperçu structuré de l'architecture et des technologies utilisées dans le développement des applications Mangwele et Mavimpi. En suivant ces conventions et normes, nous assurons des applications robustes, maintenables et facilement extensibles pour répondre aux besoins de la communauté.
