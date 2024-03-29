---
title: Hyphenation - automatyczne dzielenie wyrazów na sylaby
date: 2019-04-10 16:00:34.000000000 +02:00
identifier: hyphenation
categories:
- Frontend
- Na dłużej
- Polski
tags:
- angular
- code
- hyphenate
- pipe
permalink: "/2019/04/automatyczne-dzielenie-wyrazow-na-sylaby-hyphenation/"
excerpt: Aplikacje tłumaczone na niektóre języki (m. in. skandynawskie, ale także
  niemiecki) muszą poradzić sobie z poprawnym wyświetlaniem wyrazów dłuższych niż
  niejedno zdanie :) Jak to zrobić? Dowiecie się już dziś!
---
<p>Wykonując projekty dla klienta niemieckiego spotkałem się z problemem zgodnego z gramatyką dzielenia długich wyrazów na sylaby. Ma to znaczenie, ponieważ słowa takie jak “Datenschutzerklärung”, “Zusammenfassung” czy “Änderungswunsch” nie mieszczą się w jednej linii na urządzeniach mobilnych. Strona, która nie potrafi poprawnie wyświetlać takiego tekstu jest, w odczuciu użytkowników, przygotowana nieprofesjonalne, a rozwiązaniem tego problemu jest implementacja zgodnego z gramatyką dzielenia długich wyrazów na sylaby (ang. <em>hyphenation</em>). Zaczynajmy!</p>
<h2>Zarys rozwiązania</h2>
<p>Chcemy, aby rozwiązanie pozwalało włączyć mechanizm dla wybranych słów bądź zdań bezpośrednio w szablonie HTML strony, najlepiej z możliwością wskazania dodatkowych opcji, np. dzielenie na sylaby tylko N najdłuższych słów w zdaniu bądź dzielenie tylko wyrazów dłuższych niż M znaków. Mając na względzie te założenia oraz wybraną technologię (Angular), logicznie decydujemy się na użycie narzędzia pipe. Dzięki temu użycie tego mechanizmu będzie wyglądać np. tak:</p>
<pre>
    &lbrace;&lbrace; 'Datenschutzerklärung' | hyphenate &rbrace;&rbrace;
    &lt;!-- wyświetla Da•ten•schutz•er•klä•rung -->
</pre>
<p>gdzie • oznacza specjalną encję &amp;shy­­ normalnie nie będącą wyświetlaną przez przeglądarki, ale definiującą punkty, w których może ona dokonać podziału na sylaby w razie potrzeby.</p>
<p>Aby to uzyskać skorzystamy z biblioteki hypher (<a href="https://github.com/bramstein/hypher" target="_blank">https://github.com/bramstein/hypher</a>), służącej właśnie do dzielenia słów zgodnie z gramatyką wybranego języka (ang. <em>hyphenation</em>).</p>
<h2>Czym jest Pipe w “nowym” Angularze?</h2>
<p>Zanim omówię implementację, krótka informacja nt. narzędzia pipe. Umożliwia ono transformację wartości wejściowej w zdefiniowany przez nas sposób, otrzymując nową wartość na wyjściu, gotową do wyświetlenia w szablonie. Aby stworzyć własny pipe wystarczy zadeklarować nową klasę implementującą interfejs PipeTransform z funkcją transform. W samym frameworku zdefiniowane są pewne standardowe transformacje, np. CurrencyPipe czy DatePipe, które można wykorzystywać&nbsp;w następujący sposób:</p>
<pre>
    &lbrace;&lbrace; 1.23 | currency }}
    &lt;!-- wyświetla '$1.23' -->

    &lbrace;&lbrace; dateObj | date:'medium' }}
    &lt;!-- wyświetla 'Jan 27, 2019, 10:40:42 PM' -->
</pre>
<p>Nasza implementacja również będzie wykorzystywała narzędzie pipe.</p>
<h2>Programistyczne mięcho!</h2>
<h3>Instalacja niezbędnych paczek</h3>
<p>Zaczynamy od instalacji biblioteki hypher wraz z potrzebnymi wzorcami językowymi (niemiecki w tym przykładzie):</p>
<pre>
    npm install hypher hyphenation.de
</pre>
<h3>Szkielet klasy HyphenatePipe</h3>
<p>Następnie tworzymy plik zawierający definicję naszego pipe’a, np. hyphenate.pipe.ts, w którym na początku importujemy bilbiotekę i wzorzec językowy:</p>
{% highlight typescript %}
import * as Hypher from 'hypher';
import * as german from 'hyphenation.de';
{% endhighlight %}
<p>Zaraz za tym możemy rozpocząć definiowanie klasy:</p>
{% highlight typescript %}
/**
 * Hyphenates given text, based on Hypher library
 * @example
 *  'Finanzierungsanfrage' | hyphenate
 *  formats to: 'Fi-nan-zie-rungs-an-fra-ge'
 *  (with ­ entities in place of hyphens)
 */
@Pipe({name: 'hyphenate'})
export class HyphenatePipe implements PipeTransform {
{% endhighlight %}
<p>Dekorator @Pipe mówi kompilatorowi jaki typ klasy chcemy zaimplementować. W nawiasie podajemy obiekt konfiguracyjny z kluczem name, oznaczającym nazwę, po której będziemy odwoływać się w szablonach do pipe’a. Ważne jest też zaznaczenie implementacji interfejsu PipeTransform, zawierającego wspomnianą metodę transform. Na samej górze dodałem przydatne komentarze w stylu jsdoc, o których napisałem osobny <a href="/2019/03/jsdoc/" target="_blank" rel="noreferrer noopener" aria-label="wpis (otwiera się na nowej zakładce)">wpis</a> na blogu :)</p>
<p>Następnie mamy sekcję inicjalizacyjną:</p>
{% highlight typescript %}
private hyphenator: Hypher = null;
private hyphenChar = '\u00AD';

constructor() {
  this.hyphenator = new Hypher(german);
}
{% endhighlight %}
<p>Tu słowo wyjaśnienia: aby móc korzystać z biblioteki hypher, musimy stworzyć obiekt klasy Hypher z parametrem wzorca językowego. Stąd pole obiektu o nazwie hyphenator. Natomiast ‘\u00AD’ jest znakiem “miękkiego” łącznika (encja &amp;shy­). Jest to znak niewyświetlany normalnie przez przeglądarkę, ale zaznaczający miejsce, w którym może ona dokonać podziału na sylaby w razie potrzeby.</p>
{% highlight typescript %}
/**
 * Hyphenates given text
 * @param {string} text Text to hyphenate
 * @param {HyphenateOptions} options
 *  Optional. Additional options can be specified here.
 */
transform(text: string, options: HyphenateOptions = {}): string {
{% endhighlight %}
<p>Najważniejszą metodą klasy jest transform, która to posiada 2 parametry: value typu string, czyli wartość wejściową dla pipe’a, która będzie przekształcana oraz options typu HyphenateOptions, czyli opcje definiujące dodatkowe zachowanie.</p>
<h3>Dodatkowe opcje pipe'a - interfejs HyphenateOptions</h3>
{% highlight typescript %}
/**
 * @desc Options which can be given into hyphenate pipe
 * @prop {number} onlyNLongest Hyphenate only N longest words from given text
 * @prop {number} longerThan Hyphenate only words longer than N characters
 */
interface HyphenateOptions {
  onlyNLongest?: number;
  longerThan?: number;
}
{% endhighlight %}
<p>Oznacza to, że nasz pipe będzie miał możliwość ograniczenia mechanizmu dzielenia na sylaby tylko do wyrazów dłuższych niż M znaków oraz dla tylko N najdłuższych wyrazów w zdaniu. Opcje te będzie można przekazywać do pipe’a za pomocą konstrukcji value | hyphenate:{ longerThan: M, onlyNLongest: N }.</p>
<h3>Punkt kulminacyjny, czyli implementacja metody transform</h3>
{% highlight typescript %}
transform(text: string, options: HyphenateOptions = {}): string {
  const words = text.split(/\s+/);
  const hyphenateNLongest = Math.min(
    words.length, options.onlyNLongest || words.length
  );
  const hyphenateLongerThan = options.longerThan || 0;
  const wordsToHyphenate = words
   .concat()
   .sort((word1, word2) => word2.length - word1.length)
   .slice(0, hyphenateNLongest)
   .filter(word => word.length > hyphenateLongerThan);
  return words
    .map(word => {
      if (wordsToHyphenate.indexOf(word) !== -1) {
        return this.hyphenator.hyphenate(word).join(this.hyphenChar);
      }
      return word;
    })
    .join(' ');
}
{% endhighlight %}
<p>Ponieważ przekazanie dodatkowych opcji nie jest obowiązkowe, domyślnie dzielone na sylaby są wszystkie wyrazy. Osiągane jest to poprzez ustawienie opcja onlyNLongest na równą liczbie słów a longerThan na 0. </p>
<p>Poprzez użycie metody concat na tablicy words tworzymy jej kopię, którą następnie sortujemy. Jest to potrzebne, ponieważ operacja sortowania zmienia kolejność elementów oryginalnej tablicy. Następnie wybieramy z niej tylko N najdłuższych wyrazów a ostatecznie filtrujemy tak, aby na końcu znalazły się tylko wyrazy dłuższe niż M znaków. Sam proces dzielenia na sylaby (główna instrukcja return) polega na utworzeniu nowej tablicy na podstawie tablicy słów z wejścia (words). W kroku tym (operacja .map(word =&gt; …) sprawdzamy czy powinniśmy podzielić dane słowo (czy znajduje się w tablicy wordsToHyphenate). Jeśli tak to wywołujemy metodę hyphenate na obiekcie this.hyphenator (czyli obiekcie bibliotecznym), a otrzymaną tablicę sylab łączymy znakiem “miękkiego” łącznika (ang. <em>soft hyphen</em>, encja &amp;shy­). Jeśli słowo nie powinno zostać podzielone to zwracamy je bez transformacji (return word). Ostatnim krokiem jest połączenie tablicy słów na pomocą spacji (.join(‘ ‘)).</p>
<h2>Podsumowanie</h2>
<p>Myślę, że przedstawione przeze mnie rozwiązanie jest całkiem eleganckie i również Wam przypadnie do gustu. Tym bardziej, że sytuacje, w których może się przydać, z mojego doświadczenia, nie są wcale takie rzadkie. Wbudowana w Angular możliwość definiowania własnych pipe’ów sprawdza się tutaj idealnie. Dodatkowe, opcjonalne parametry, które możemy przekazać pozwalają na stworzenie rozwiązania elastycznego i rozszerzalnego. Dzięki, że dobrnęliście aż do końca, miłego dnia wszystkim!</p>
