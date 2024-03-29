---
title: Mechanizm uwierzytelniania na serwerach przedprodukcyjnych
date: 2019-04-24 16:00:10.000000000 +02:00
identifier: authentication-on-preprod-servers
categories:
- Bezpieczeństwo
- DevOps
- Polski
tags:
- basic auth
- bezpieczeństwo
- cookie
- nginx
- apache
permalink: "/2019/04/mechanizm-uwierzytelniania-na-serwerach-przedprodukcyjnych/"
excerpt: Jak prosto i skutecznie ograniczyć dostęp do naszego serwera testowego/przedprodukcyjnego?
  Zapraszam na garść szybkich pomysłów!
---
<p>Rozpoczynając nowy projekt często pojawia się potrzeba zorganizowania osobnych instancji naszej aplikacji, dostępnych wyłącznie dla testerów bądź klientów końcowych. Z założenia mają one służyć testowaniu i sprawdzaniu <del>wypocin</del> efektów naszej wielogodzinnej i oddanej pracy. Właśnie wtedy przyda się nam wiedza w jaki sposób ograniczyć dostęp do takich przed-produkcyjnych serwerów w prosty i skuteczny sposób. Potrzebny nam jest <strong>mechanizm uwierzytelniania użytkowników</strong> w dostępie do strony.</p>
<h2>"Basic Access Authentication"</h2>
<p>Czyli, prościej mówiąc, wyskakujące okienko z pytaniem o nazwę użytkownika i hasło, gdy tylko próbujemy wejść pod dany adres. Wydaje się zbyt proste i piękne, aby było prawdziwe? Cóż, rzeczywiście ten mechanizm uwierzytelniania posiada pewien haczyk. Mianowicie dane (tj. nazwa użytkownika i hasło) przesyłane są przez przeglądarkę w formie zakodowanej w base64. I nic więcej. Oznacza to, że ten sposób <strong>wymaga włączenia przez nas obsługi protokołu HTTPS i każdorazowe przekierowywanie żądań HTTP na HTTPS</strong>. Dlaczego? Ponieważ dane przesyłane poprzez HTTP nie są w żaden sposób chronione, a samo kodowanie base64 jest banalne do odwrócenia. </p>
<p>Istnieje <a href="https://www.base64decode.org/" target="_blank">wiele</a> stron, na których możemy samodzielnie zakodować oraz odkodować dane w tym formacie. Z zalet z kolei możemy jednak wymienić to, że, dla naszej wygody, przeglądarka zapamiętuje wpisane dane. Ma to znaczenie, ponieważ, jeśli były poprawne, wysyła je automatycznie przy kolejnych żądaniach uwierzytelnienia. Dzięki temu nie będziemy musieli wpisywać ich za każdą próbą dostępu do serwera. Dodatkowo, możemy zdefiniować wiele par &lt;nazwa użytkownika, hasło&gt; i nadawać im dostępy np. tylko do niektórych elementów strony.</p>
<h3>Stworzenie pliku z listą użytkowników i ich haseł</h3>
<p>Zacząć należy od stworzenia odpowiedniego pliku (nazywanego zwyczajowo .htpasswd), zawierającego definicje par &lt;nazwa użytkownika, hasło&gt;, którym zezwalamy na dostęp do strony. Najprościej wygenerować go za pomocą narzędzia htpasswd, dostępnego na Ubuntu w pakiecie apache2-utils:</p>
<pre>
    sudo apt-get install apache2-utils
</pre>
<p>Teraz możemy stworzyć odpowiedni plik wraz z pierwszym użytkownikiem:</p>
<pre>
    htpasswd -c .htpasswd nitrooos
</pre>
<p>Zostaniemy poproszeni o wpisanie hasła dla nowego użytkownika i jego powtórzenie. Po zatwierdzeniu wpis dla użytkownika nitrooos będzie gotowy :) Kolejnych użytkowników można doadć poprzez ponowne wykonanie powyższej komendy, ale bez przełącznika -c. Odpowiada on bowiem za utworzenie nowego pliku o podanej nazwie. Przykładowa zawartość pliku .htpasswd wygląda następująco:</p>
<pre>
    nitrooos:$apr1$4gKNcA2h$rhk7RtslHU4m.6EAs6T7j.
    admin:$apr1$pLzt4H94$R6wTAUQphdWjvavOphViX.
</pre>
<h3>Włączamy mechanizm uwierzytelniania w konfiguracji serwera sieciowego</h3>
<p>Kolejnym krokiem jest włączenie mechanizmu w konfiguracji serwera HTTP. Dla każdego z nich robi się to inaczej, niemniej jednak pokażę przykładowe opcje konfiguracje dla Apache oraz nginx. W przypadku serwera Apache należy w pliku konfiguracji odpowiedniego wirtualnego hosta dodać sekcję &lt;Directory&gt; ze wskazaniem ścieżki, pod którą rezyduje plik .htpasswd (/etc/apache2/.htpasswd w tym przykładzie):</p>
{% highlight apache %}
<VirtualHost *:80>
    DocumentRoot /var/www/html

    <Directory "/var/www/html">
        AuthType Basic
        AuthName "Restricted Content"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>
</VirtualHost>
{% endhighlight %}
<p>W przypadku serwera nginx należy do odpowiedniej sekcji location dodać wpisy auth_basic oraz auth_basic_user_file. Plik .htpasswd znajduje się w tym przypadku pod ścieżką /etc/nginx/conf.d/.htpasswd:</p>
{% highlight nginx %}
location / {
    auth_basic "Restricted content";
    auth_basic_user_file /etc/nginx/conf.d/.htpasswd;
}
{% endhighlight %}
<p>Po dokonaniu tych zmian serwer należy oczywiście zrestartować:</p>
<pre>
    sudo service apache2 restart
</pre>
<p>lub też:</p>
<pre>
    sudo service nginx restart
</pre>
<p>Jeśli wszystko poszło po naszej myśli, w tym momencie dostęp do naszej witryny będzie chroniony przez mechanizm uwierzytelniania Basic Access Authentication. </p>
<h4>Przekierowanie ruchu HTTP na HTTPS</h4>
<p>Jak napisałem na początku, metoda ta wymaga obsługi ruchu poprzez protokół HTTPS. Oznacza to konieczność zdobycia odpowiedniego certyfikatu oraz włączenie go w konfiguracji serwera sieciowego. Nie jest to tematem tego wpisu, należy jednak pamiętać o stworzeniu przekierowania ruchu HTTP na HTTPS (przykład dla serwera nginx):</p>
{% highlight nginx %}
server {
    listen [::]:80;
    listen 80;
    rewrite ^ https://$host$request_uri? permanent;
}
{% endhighlight %}
<h2>Bonus: ciastko sesyjne jako podstawowy mechanizm uwierzytelniania</h2>
<p>Alternatywą dla powyższej metody może być ustawianie w przeglądarce użytkownika specjalnego ciastka sesyjnego. Taki plik (ang. <em>cookie</em>) o ustalonej wartości, wymagany będzie aby uzyskać dostęp do serwisu. Bezpieczniejsza wersja zakłada, że użytkownik samemu ustawia sobie takie ciastko, np. w kosoli JavaScript przeglądarki, wersja prostsza to po prostu udostępnienie pod specjalnym adresem strony ustawiającej ciastko za nas. Oczywiście ta metoda nie wyklucza stosowania poprzedniej. Na potrzeby prezentacji załóżmy, że ciastko uwierzytelniające, wymagane przez serwer posiada nazwę "TestServerAuthCookie" i wartość "test_server_granted".</p>
<h3>Ustawianie ciastka sesyjnego przez użytkownika w konsoli JavaScript</h3>
<p>W tym scenariuszu użytkownik, aby uzyskać dostęp do strony, musi po wejściu pod jej adres uruchomić narzędzia deweloperskie przeglądarki (np. DevTools w Google Chrome), przejść do zakładki "Console" i wkleić prosty kawałek kodu:</p>
{% highlight javascript %}
document.cookie = "TestServerAuthCookie=test_server_granted";
{% endhighlight %}
<p>Oczywiście w rzeczywistej implementacji nazwa i wartość ciastka nie powinny być tak przewidywalne i łatwe do odgadnięcia. Do tego dochodzi jeszcze konieczność odpowiedniego skonfigurowania serwera sieciowego tak, aby sprawdzał czy wymagane ciastko jest obecne w parametrach żądania HTTP. W przypadku serwera Apache odpowednie reguły wyglądają tak:</p>
{% highlight apache %}
RewriteCond %{HTTP_COOKIE} !TestServerAuthCookie=test_server_granted
RewriteRule .* - [NC,L,F]
{% endhighlight %}
<p>Natomiast dla nginx'a są następujące:</p>
{% highlight nginx %}
location / {
    if ($http_cookie !~ 'TestServerAuthCookie=test_server_granted') {
        return 403;
    }
}
{% endhighlight %}
<p>Oznacza to, że próbując uzyskać dostęp do naszego serwisu otrzymamy status błędu 403 Forbidden za każdym razem, gdy w żądaniu nie znajdzie się ciastko o pożądanej nazwie i wartości.</p>
<h3>Specjalny link uwierzytelniający</h3>
<p>Zwróćmy uwagę, że podany powyżej sposób może być dość niewygodny. Wymaga on otwierania konsoli przeglądarki za każdym razem, gdy potrzebujemy się uwierzytelnić na serwerze. Może być to uciążliwe np. dla osób testujących naszą aplikację, ponieważ często korzystają oni z trybu incognito, który to stanowi nową, osobną sesję. W związku z tym ustawianie ciastka sesyjnego wymagane jest po każdorazowej inicjalizacji trybu incognito. Aby nieco ułatwić życie takim <del>maruderom</del> osobom, możemy stworzyć osobny URL ustawiający wymagane ciastko w przeglądarce automatycznie. Rozwiązanie to jest o tyle wygodniejsze, że link taki można zapamiętać i ustawić np. na pasku zakładek przeglądarki. Dzięki temu proces kontroli dostępu sprowadzi się do jednego kliknięcia w przycisk. Prościej już się nie da :)</p>
<p>Stwórzmy więc osobną ścieżkę na serwerze i zwróćmy z niej dokument HTML, zawierający skrypt ustawiający wymaganą wartość ciastka. Zacząć należy od utworzenia pliku site_login.html, zawierającego prosty skrypt JavaScript, ustawiający ciastko sesyjne. Plik ten powinien wyglądać np. tak:</p>
{% highlight html %}
<html>
    <body>
        <script>
            document.cookie = 'TestServerAuthCookie=test_server_granted';
            window.location.replace('http://test.our.gretest.web.service');
        </script>
    <body>
<html>
{% endhighlight %}
<p>Następnie należy zdefiniować w konfiguracji serwera regułę, która w przypadku wejścia pod specjalny adres (link uwierzytelniający) dokona przekierowania pod /site_login.html. W przypadku serwera Apache konfiguracja wygląda tak:</p>
{% highlight apache %}
RewriteCond "%{REQUEST_URI}" "!=/authentication_link_QRSvD44xNedyEeqmyGWtevidLbmeUG1NGaeVeEtJ.html"
RewriteCond %{HTTP_COOKIE} !TestServerAuthCookie=test_server_granted
RewriteRule .* /site_login.html [NC,L,F]
{% endhighlight %}
<p>Natomiast dla nginx'a prezentuje się następująco:</p>
{% highlight nginx %}
location /authentication_link_QRSvD44xNedyEeqmyGWtevidLbmeUG1NGaeVeEtJ.html {
    add_header X-Robots-Tag "noindex, nofollow, nosnippet, noarchive";
    return 200 /site_login.html;
}
{% endhighlight %}
<p>Jeśli wejdziemy pod specjalny adres (link uwierzytelniający, tutaj /authentication_link_QRSvD44xNedyEeqmyGWtevidLbmeUG1NGaeVeEtJ.html), serwer zwróci prosty dokument HTML. Skrypt w nim zawarty automatycznie ustawi ciastko sesyjne na pożądaną wartość, po czym dokona przekierowania (window.location.replace) na stronę główną naszego serwisu. Proste i wygodne. Dodatkowo ustawiamy nagłówek odpowiedzi X-Robots-Tag na wartość "noindex, nofollow, nosnippet, noarchive". Sprawi to, że roboty indeksujące, nawet jeśli zbłądzą i odnajdą nasz link, nie zaindeksują go w żaden sposób.</p>
<h2>Podsumowanie</h2>
<p>Pokazałem kilka pomysłów, jak może być zorganizowany mechanizm uwierzytelniania użytkowników w dostępie do serwerów, na których serwujemy wersje deweloperskie/testowe/przedprodukcyjne naszych aplikacji. Są to oczywiście rady dość proste, które jednak mogą stanowić dobry punkt wyjścia w temacie uwierzytelniania użytkowników serwerów testowych. Dla osób chcących zachować większe bezpieczeństwo bądź kontrolę nad tym, kto dokładnie, kiedy może uzyskać dostęp czy też chcących logować zdarzenia dostępu do strony, mogą okazać się niewystarczające. Czasami jednak, gdy pracuje się pod presją czasu, lepsze jest takie rozwiązanie niż żadne. Natomiast dla niektórych projektów może okazać się wręcz wystarczające. Cieszę się, że mogłem podzielić się z Wami swoją wiedzą! Jendocześnie zapraszam do śledzenia bloga i czekania na kolejne wpisy, bo one już niedługo :)</p>
