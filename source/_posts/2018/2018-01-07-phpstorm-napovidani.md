---
id: 54
layout: post
title: "Jak naučit PhpStorm chápat kód"
perex: '''
Fungující napovídání syntaxe vašeho kódu je naprosto základním předpokladem pro dobré fungování pokročilých nástrojů, které vám PhpStorm nabízí. Existuje několik možností, jak PhpStormu pomoci váš kód pochopit, a v tomto článku si je postupně ukážeme.  
'''
author: 6
lang: cs
---

Fungující napovídání syntaxe vašeho kódu je naprosto základním předpokladem pro dobré fungování pokročilých nástrojů, které vám PhpStorm nabízí. Existuje několik možností, jak PhpStormu pomoci váš kód pochopit. Začneme těmi základními a postupně se dostaneme až k pokročilým. 

Nástroje jako refaktoring a inspekce kódu jsou plně závislé na tom, jak dobře dokáže PhpStorm váš kód pochopit. Ale protože je PHP dynamicky typovaný jazyk, tak je to mnohem složitější úkol, než třeba ve staticky typované Javě.

Ať se nám to líbí nebo ne, tak spousta existujícího PHP kódu je založena na více či méně náhodných polích. Ne každý má to štěstí, že může pracovat s kódem napsaným letos pro PHP 7.2 podle DDD a naprosto striktně dodržujícím SRP. Velmi pravděpodobně se naopak setkáte s kódem, který by mohl běžet i na PHP 5.3. A někdy bohužel na produkci i běží. 

PhpStorm vám může velmi pomoct právě při správě takového legacy kódu. Ale může pracovat jen s tím, co mu dáte. A teď si ukážeme, jak mu dát těch informací co nejvíc. 

## Popisování parametrů volání a návratových hodnot

### Docblocky

Docblocky jsou dokumentační komentáře. Nejsou přímo parsovány při zpracování souboru, ale lze k nim přistoupit z kódu aplikace pomocí reflexe nebo pomocí externích nástrojů (jako třeba IDE). Běžný docblock vypadá nějak takto:  

```php
<?php
/**
 * @param string $name
 * @param int $age
 * @param Address|null $address
 * @return User
 */
public function createUser($name, $age, $address)
{
    return new User($name, $age, $address);
}
```


Blok výše říká IDE, že parametr `$name` je `string`, `$age` je `integer` a `$address` je buď instance `Address` nebo `null`. Také tím říkáme, že metoda vrací instanci `User`. I když v tomto konkrétním případě by PhpStorm byl schopný odhadnout návratový typ `User` sám, mnohdy to není možné. 

Všimněte si, že v případě adresy povolujeme jak `Address` tak `null` - definuje se to pomocí svislítka (`|`). Je důležité popisovat všechny existující možnosti. V tomto případě nás díky tomu PhpStorm upozorní, že máme zkontrolovat, jestli není adresa prázdná, kdykoli voláme něco jako `$address->getZipCode()`. Přitom stále funguje napovídání metod třídy `Address`. 


Docblocky jsou skvělý nástroj pro starší verze PHP. Pro moderní verze PHP však existuje nástroj ještě lepší. 

### Deklarace typů

Od PHP 7.1 (pokud se obejdete bez nullable, tak již od 7.0) je možné výše zmíněný kus kódu přepsat do následující podoby: 

```php
<?php
declare(strict_types=1);
// ...
public function createUser(string $name, int $age, ?Address $address): User
{
    return new User($name, $age, $address);
}
```

Tato konstrukce má úplně ten samý význam, ale místo komentářů využívá přímo konstrukce jazyka. Díky tomu jsou typy vynuceny, když funkci použijeme. Pokud je `$age` definováno jako integer, tak si můžeme být jisti, že to opravdu interger bude. Není třeba žádné další validace. Samotné PHP by vyhodilo `TypeError` pokud bychom tam poslali cokoli jiného. Jen je třeba se ujistit, že máte v souboru [přidanou  `strict_types` deklaraci](http://php.net/manual/en/functions.arguments.php#functions.arguments.type-declaration.strict), která vypne přetypovávání. V opačném případě vám PHP s klidem převede `"11 horses"` na `11` [jako normálně](https://3v4l.org/QlLOV) (pro porovnání chování se [strict types](https://3v4l.org/bUAEr)).

## Union typy a pole

Union typ je typ, který se skládá z více dalších typů. Představme si například třídu, která pracuje s datumem a v konstruktoru přijímá všechny možné formáty (string, unix timestamp nebo instanci DateTime)

```php
<?php 
function __construct($date) { /* ... */ }
```
 
V tomhle případě nemůžete uvést jako datový typ proměnné `$date` prostě `DateTimeInterface|string|int`. [Alespoň zatím ne](https://wiki.php.net/rfc/union_types). Je potřeba se vrátit zpět k docblockům:  

```php
<?php
/**
 * @param DateTimeInterface|string|int $date
 */
function __construct($date) { /* ... */ }
```

Dalším případem, kde je třeba návrat k docblockům, jsou generika a pole objektů. Databázový dotaz může například vrátit kolekci uživatelů (`Collection`). Pomocí typové deklarace můžeme napsat

```php
<?php
function getUsers(): Collection {}
```

Ale to nepostihne informaci o tom, že uvnitř kolekce jsou uživatelé. Takže nebude fungovat doplňování pro `$collection->first()->???`. Opět se musím vrátit k docblockům. 

```php
<?php
/**
 * @return User[]|Collection
 */
function getUsers(): Collection {}
```

Tímto způsobem získáme doplňování jak pro `$collection->count()`, tak pro `$collection->first()->getUsername()`. 


## Co když se typy mění za běhu? 

Továrny a service lokátory vrací různé typy podle toho, s jakým parametrem je zavoláme. Podívejme se na následující kód:

```php
<?php
// není jasné, jakého bude $logger typu
$logger = $container->get('LoggerInterface');
$logger->???
```

Můžeme však napovědět přímo v kódu dokumentačním komentářem.  

```php
<?php
/** @var LoggerInterface $logger */
$logger = $container->get('LoggerInterface');
$logger->log(/* code completion */);
```

Tento způsob je široce podporovaný a mnoho nástrojů ho dokáže využívat. Ale nezapomeňte, že se stále jedna o přístup založený na komentářích a je tedy snadno možné, že _se při refaktoringu rozuteče oproti kódu a už ho nikdy nikdo neupraví_. 

## A co magické metody?  

```php
<?php
/**
 * @property-read $username
 * @property $name
 * @property-write $password
 * @method void reset()
 * @method static Config factory()
 */
class Config {
    private $config = [
        'username' => 'john', 
        'name' => 'John Doe', 
        'password' => '123456'
    ];
    
    public function __get($property){
        if ($property === 'password') {
            return null; // you can't read password
        }
        return $this->config[$property];
    }
    
    public function __set($property, $value){
        if ($property === 'username') {
			return null; // you can't set username
		}
		return $this->config[$property];
    }
    
    public function __call($method, $arguments){
        if ($method === 'reset') {
            $this->config = [];
        }
    }
    
    public static function __callStatic($method, $arguments){
        if ($method === 'factory') {
            return new self();
        }
    }
}
```

U třídy výše (přiznávám, je to extrémní hovnokód) můžete vidět jednotlivé možnosti, jak se vypořádat s magickými metodami. 

* `@property-read` - znamená, že existuje veřejný atribut, který lze číst. Napovídání pro `$username = $config->username;` bude fungovat, ale pokud zkusíte do proměnné zapsat, PhpStorm vám to označí jako chybu. 
* `@property-write` - znamená, že existuje veřejný atribut, do kterého lze zapisovat. Napovídání `$config->password = 'dummy';` bude fungovat, ale pokud se pokusíte atribut přečíst, PhpStorm to označí jako chybu. 
* `@property` - kombinuje výše zmíněné (čtení a zápis)
* `@method` - znamená, že existuje veřejná metoda a specifikuje její návratový typ. V tomto případě vám PhpStorm napoví `$config->reset()`. 
* `@method static` - znamená, že existuje veřejný statická metoda. V tomto případě `Config::factory()` Tím pádem bude následně fungovat například napovídání v případě `Config::factory()->reset()`.  

## PhpStorm meta file

Na konec jsem si nechal takovou specialitku - [soubor `.phpstorm.meta.php`](https://confluence.jetbrains.com/display/PhpStorm/PhpStorm+Advanced+Metadata). 

```php

<?php
// in .phpstorm.meta.php\myframework.meta.php
namespace PHPSTORM_META {
  override(\ServiceLocator::get(0),
    map([
      'foo' => \FooInterface::class, // když zavolám get('foo'), dostanu FooInterface
      \ToTownInterface::class => \User::class, // když zavolám get(\ToTownInterface::class), dostanu User
      // když zavolám get('AnythingElse'), dostanu AnythingElse (výchozí chování) 
    ])
  );
  
  override(\IteratorGenerator::get(0),
    map([
      // "@" je nahrazen čímkoli, co pošlete jako parametr  
      '' => '@Iterator|\Iterator' // když zavolám get('User'), dostanu UserIterator|Iterator
    ])
  );
}
```

Bohužel je v současnosti možné takto specifikovat jen první parametr volání. Je to čistě omezení současné implementace v PhpStormu. Nicméně [z definice](https://github.com/JetBrains/phpstorm-stubs/blob/master/meta/.phpstorm.meta.php) je vidět, že samotný formát je na to připraven a implementaci je možné do budoucna rozšířit. 

## Závěr

V článku jsme si ukázali různé možnosti, jak PhpStormu napovědět, jakého typu je proměnná nebo co vrací která metoda. V čím lepším stavu bude v tomto ohledu váš kód, tím bezpečněji můžete využívat přejmenovávání a refaktoringy v PhpStormu. Pokud chcete, tak můžete kontrolu typů přidat do svého coding standardu pomocí [`SlevomatCodingStandard.TypeHints.TypeHintDeclaration`](https://github.com/slevomat/coding-standard). 

Napadá vás ještě nějaký další způsob, jak napovídat typy, nebo jsem na něco zapomněl? Napište mi, nebo rovnou pošlete k článku pullrequest. 
