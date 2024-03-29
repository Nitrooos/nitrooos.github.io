---
layout: post
title: HyphenatePipe - uzupełnienie do ostatniego wpisu
date: 2019-05-15 20:00:54.000000000 +02:00
identifier: hyphenation-appendix
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Bezpieczeństwo
tags: []
meta:
  _edit_last: '1'
  _yoast_wpseo_primary_category: '1'
  wpbf_options: a:1:{i:0;s:13:"layout-global";}
  wpbf_sidebar_position: global
  _yoast_wpseo_content_score: '90'
  _yoast_wpseo_focuskw: HyphenatePipe
  _yoast_wpseo_metadesc: Uzupełnie wpisu nt. klasy HyphenatePipe - co zrobić gdy używanie
    pipe'a w aplikacji pogarsza jej wydajność? Jak działa Pipe w Angularze? Zapraszam!
  _yoast_wpseo_linkdex: '78'
  _wp_old_slug: hyphenationpipe-uzupelnienie
author:
  login: bartoszkostaniak_k9gtfpdt
  email: bartoszkostaniak@gmail.com
  display_name: nitrooos
  first_name: ''
  last_name: ''
permalink: "/2019/05/hyphenatepipe-uzupelnienie/"
excerpt: Uzupełnie wpisu nt. klasy HyphenatePipe - co zrobić gdy używanie pipe'a w
  aplikacji pogarsza jej wydajność? Jak działa Pipe w Angularze? Zapraszam!
---
<p>W jednym z moich niedawnych wpisów (<a target="_blank" href="/2019/04/automatyczne-dzielenie-wyrazow-na-sylaby-hyphenation/">Hyphenation - automatyczne dzielenie wyrazów na sylaby</a>) poruszyłem problem zgodnego z gramatyką dzielenia długich wyrazów na sylaby w aplikacji. Cel ten został osiągnięty poprzez zaimplementowanie własnego pipe'a, czyli klasy HyphenatePipe. Uznałem, że wpis ten wymaga uzupełnienia, ponieważ podana w nim implementacja zawiera pewne niedopowiedzenie. Spokojnie, nie jest to krytyczna podatność, ale okazuje się, że w pewnych okolicznościach może napsuć nam nerwów ;) O co chodzi? Zapraszam do przeczytania krótkiego uzupełnienia!</p>
<p><strong>TLDR;</strong><br />Operacja tworzenie Pipe'a ze wspomnianego wpisu (HyphenatePipe) jest czaso- i pamięciochłonna. Przy intensywnym używaniu tego narzędzia Angular będzie (potencjalnie) wielokrotnie potwarzał tę kosztowną operację, zupełnie niepotrzebnie. Potrzebna jest modyfikacja przedstawionej implementacji tak, aby tworzenie instancji HyphenatePipe było znacznie mniej kosztowne.</p>
<h2>W jaki sposób w Angularze działa narzędzie Pipe?</h2>
<p>Za <a href="https://angular.io/guide/pipes" target="_blank">dokumentacją</a>, w Angularze rozróżnia się dwa podstawowe rodzaje Pipe'ów:</p>
<ul>
  <li><em>pure pipe</em>, czyli odmiana domyślna, w założeniu nie posiada wewnętrznego stanu, dlatego Angular stara się wykorzystać <strong>tę samą instancję</strong> w jak największej liczbie miejsc, w ramach jednego widoku. Oznacza to oszczędność pamięci oraz, co ważne w naszym przypadku, czasu potrzebnego na stworzenie kolejnych instancji</li>
  <li><em>impure pipe</em> to z kolei rodzaj, który możemy nadać dla własnego Pipe'a poprzez użycie flagi { pure: false } w dekoratorze @Pipe. Odmiana ta może posiadać wewnętrzny stan a Angular tworzy nową instancję dla każdego wystąpienia Pipe'a w widoku. Jest to opcja znacznie bardziej pamięcio- i czasochłonna, dlatego powinniśmy używać jej z rozmysłem.</li>
</ul>
<h3>Definicja problemu</h3>
<p>Pipe, który opisałem (HyphenatePipe) co prawda należy do tego pierwszego rodzaju, jednak bardzo łatwo przeoczyć pewien fakt.  <strong>Używając Pipe'a w ramach elementów potomnych </strong>(nawet jeśli są częścią tej samej strony aplikacji i są typu <em>pure</em>) <strong>zostaną utworzone osobne instancje Pipe'a</strong>. Każda z nich odpowiadać będzie pojedynczemu użyciu w widoku. Jakie ma to znaczenie? Aby to zrozumieć należy zastanowić się...</p>
<h2>Czym właściwie zajmuje się konstruktor klasy HyphenatePipe?</h2>
<p>Tworzy on nową instancję obiektu klasy Hypher (pole o nazwie hyphenator). Operacja ta wymaga zaimportowania odpowiedniego wzorca językowego do dzielenia wyrazów, a także parsowania go. W przypadku opisywanego języka (niemiecki) wzorzec ten jest bardzo duży (ok 70kB danych), w ogólności mogą mieć one od kilku do kilkudziesięciu kB. W połączeniu z tworzeniem instancji klasy HyphenatePipe dla każdego użycia konstrukcji " ... | hyphenate" w kodzie, opóźnia to ładowanie naszej aplikacji :/</p>
<p>Niestety, do tego stopnia, że problem ten okazał się być głównym czynnikiem wydłużającym czas ładowania aplikacji. W moim przypadku Pipe tworzony był 8 razy, przy czym za każdym razem wykonywana była operacja importu wzorca językowego. Dokładnie o 7 razy za dużo &lt;facepalm&gt;.</p>
<h2>Rozwiązanie problemu z klasą HyphenatePipe</h2>
<p>Rozwiązaniem jest oczywiście wykonanie importu wzorca językowego tylko 1 raz. Możemy to osiągnąć poprzez wydzielenie operacji tworzenia obiektu hyphenator klasy Hypher do dedykowanego serwisu. W naszym przypadku będzie on nazywał się HyphenationPatternsService. Implementacja jest prosta, spokojnie można przedstawić ją w jednym fragmencie kodu:</p>
{% highlight typescript %}
import { Injectable } from '@angular/core';
import * as Hypher from 'hypher';
import * as german from 'hyphenation.de';

@Injectable()
export class HyphenationPatternsService {
  private hyphenator: Hypher = null;

  constructor() {
    this.hyphenator = new Hypher(german);
  }

  public hyphenate(word: string) {
    return this.hyphenator.hyphenate(word);
  }
}
{% endhighlight %}
<p>Następnie możemy wstrzyknąć&nbsp;ten serwis jako zależność do klasy HyphenatePipe:</p>
{% highlight typescript %}
constructor(private hyphenationService: HyphenationPatternsService) { }
{% endhighlight %}
<p>Od teraz tworzenie nowych instancji HyphenatePipe jest bardzo szybkie i nie rzutuje na czasie ładowania aplikacji. Wzorzec językowy dla dzielenia wyrazu na sylaby jest ładowany tylko 1 raz. Gotowe!</p>
