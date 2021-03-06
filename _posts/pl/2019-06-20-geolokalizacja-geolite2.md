---
title: Geolokalizacja użytkowników za pomocą GeoLite2
date: 2019-06-20 20:00:04.000000000 +02:00
identifier: geolite2
categories:
- Backend
- Polski
- Python
tags:
- "#geoip"
- "#python"
- geolokalizacja
permalink: "/2019/06/geolokalizacja-geolite2/"
excerpt: Dziś na warsztacie temat lokalizacji użytkowników naszej witryny. W jaki
  sposób określić kraj (i miasto), z którego wizytowana jest strona? Odpowiedź w artykule!
---
<p>Okazuje się, że wiedza o tym, skąd pochodzą odwiedzający witrynę goście pozwala na lepsze dostosowanie jej do ich wygody, a nawet do podejmowania decyzji w kontekście biznesowym. Decyzje takie mają bezpośrednie przełożenie na wielkość ruchu i zainteresowanie naszą witryną! Jak to możliwe? Postaram się odpowiedzieć dzisiaj na to pytanie, ponieważ geolokalizacja z pomocą biblioteki GeoLite2 jest tematem tego artykułu.</p>
<p><strong>TLDR;</strong><br /><a href="https://dev.maxmind.com/geoip/geoip2/geolite2/" target="_blank">GeoLite2</a> to darmowa, solidna baza danych, zawierająca odwzorowania adresów IP na przypsane do nich kraje i miasta. Istnieje wiele gotowych interfejsów programistycznych (API) do tej bazy, m.in. dla języków C#, C, Java, JavaScript (node.js), Perl, PHP, Python i Ruby.</p>
<h2>Czym jest GeoLite2?</h2>
<p><a href="https://dev.maxmind.com/geoip/geoip2/geolite2/" target="_blank">GeoLite2</a> to baza danych (w formacie binarnym) odwzorowująca adresy IP z całego świata na państwa i miasta, z których pochodzą. Znając dany adres IP (a dostęp do niego nie jest trudny w przypadku aplikacji webowych) możemy z jej pomocą dowiedzieć się więc z jakiego państwa, a nawet miasta została odwiedzona strona. Oczywiście baza taka nie posiada (i nigdy nie będzie) stuprocentowej dokładności, dlatego też mogą zdarzyć się jej pomyłki. Z własnego doświadczenia wiem jednak, że są one niezwykle rzadkie, a sama baza działa bardzo dokładnie. Jak bardzo? Producent twierdzi, że wersja płatna (o nazwie <a href="https://www.maxmind.com/en/geoip2-services-and-databases" target="_blank">GeoIP2</a>) zawiera w sobie 99,9999% adresów IP będących w użyciu. Z kolei jej dokładność jeśli chodzi o podanie nazwy kraju, z którego pochodzi dany adres ma wynosić 99,8%. Liczby te robią wrażenie, lecz nie są podane żadne dane dotyczące wersji darmowej (czyli GeoLite2). Producent zaznacza jedynie, że wersja darmowa jest mniej dokładna i rzadziej aktualizowana.</p>
<h3>Połączenie z bazą GeoLite2</h3>
<p>Zaczynamy od pobrania bazy GeoLite2, obecnie dostępnej na stronie producenta, czyli firmy MaxMind, pod <a href="https://geolite.maxmind.com/download/geoip/database/GeoLite2-Country.tar.gz" target="_blank">tym</a> linkiem. Zauważmy, że jest to baza w wersji "country", co oznacza odwzorowanie adresów IP na nazwy państw, nie miast. Jest to mniejsza wersja darmowej bazy, istnieje jeszcze odmiana "city", odwzorowująca adresy na poszczególne miasta świata, ale jest też kilkanaście razy większa pod względem rozmiaru. </p>
<p>Następnie instalujemy jeden z dostępnych API do bazy GeoLite2 (w naszym przypadku będzie to API dla Pythona):</p>
<pre>
    $ pip install geoip2
</pre>
<p>Od teraz możemy się połączyć z bazą, najprościej za pomocą skryptu:</p>
{% highlight python %}
>>> import geoip2.database
>>> reader = geoip2.database.Reader('/path/to/GeoLite2-Country.mmdb')
{% endhighlight %}
<p>Zaznaczmy, że obiekt Reader, łączący się z bazą, służy do <em>wielokrotnego</em> wykonywania zapytań do bazy. Nie powinien być tworzony za każdym razem od nowa, ponieważ jest to operacja kosztowna. Wreszcie możemy wykonać pierwsze zapytania:</p>
{% highlight python %}
>>> response = reader.country('135.200.12.84')
>>>
>>> response.country.iso_code
'US'
>>> response.country.name
'United States'
>>> response.subdivisions.most_specific.name
'Indiana'
>>> response.subdivisions.most_specific.iso_code
'IN'
>>> reader.close()
{% endhighlight %}
<p>Otrzymujemy konkretną informację o kraju pochodzenia adresu IP i regionie (w tym przypadku stan USA). W przypadku, gdybyśmy łączyli się z bazą typu "city", moglibyśmy uzyskać dodatkowe informacje:</p>
{% highlight python %}
>>> response = reader.city('135.200.12.84')
>>>
>>> response.city.name
'Indianapolis'
>>> response.postal.code
'46226'
>>> response.location.latitude
39.7722
>>> response.location.longitude
-86.1565
>>> reader.close()
{% endhighlight %}
<p>Po zakończeniu pracy z bazą możemy wywołać metodę .close() obiektu reader (zamyka ona połączenie z bazą w sposób jawny).</p>
<p>Jako uzupełnienie dodam, że istnieje możliwość&nbsp;użycia GeoIP2 (czyli wersji płatnej) w formie web serwisu, wówczas interfejs programistyczny pozostaje niemal bez zmian, ale nie musimy trzymać&nbsp;lokalnie baz danych typu "city" ani "country".</p>
<h2>Geolokalizacja w Django</h2>
<p>Okazuje się, że (pozostając w klimacie Pythona) najpopularniejszy framework webowy dla tego języka posiada wbudowane wsparcie dla korzystania z baz GeoIP2 i GeoLite2. Odpowiednia paczka wewnątrz Django nazywa się <strong>django.contrib.gis.geoip2</strong> i umożliwia import klasy <strong>GeoIP2</strong> (odpowiednik wcześniejszego Reader'a). Klasa ta stanowi nakładkę (<em>wrapper</em>) na bibliotekę geoip2, której używa wewnętrznie. Przykładowa sesja pracy z tą paczką wygląda następująco (wzięta jako przykład ze <a href="https://docs.djangoproject.com/en/2.2/ref/contrib/gis/geoip2/" target="_blank">strony w dokumentacji Django</a>):</p>
{% highlight python %}
>>> from django.contrib.gis.geoip2 import GeoIP2
>>>
>>> geoip = GeoIP2()
>>> geoip.country('facebook.com')
{'country_code': 'US', 'country_name': 'United States'}
>>> geoip.city('72.14.207.99')
{'city': 'Mountain View',
'continent_code': 'NA',
'continent_name': 'North America',
'country_code': 'US',
'country_name': 'United States',
'dma_code': 807,
'latitude': 37.419200897216797,
'longitude': -122.05740356445312,
'postal_code': '94043',
'region': 'CA',
'time_zone': 'America/Los_Angeles'}
>>> geoip.lat_lon('salon.com')
(39.0437, -77.4875)
>>> geoip.lon_lat('uh.edu')
(-95.4342, 29.834)
>>> geoip.geos('24.124.1.80').wkt
'POINT (-97 38)'
{% endhighlight %}
<p>Widzimy, że API jest trochę zmienione, korzystanie z niego jest jednak wciąż bardzo wygodne. Dokumentacja Django służy w tym przypadku nieocenioną pomocą. Skąd jednak framework wie, gdzie znajduje się plik z bazą danych GeoIP2 (bądź GeoLite2)? Odpowiadają za to trzy stałe konfiguracyjne, które możemy ustawić w pliku <em>settings.py</em> naszego projektu:</p>
{% highlight python %}
GEOIP_PATH = os.path.join(BASE_DIR, 'common', 'geoip', 'country_dataset')
GEOIP_COUNTRY = 'Geo_db_country.mmdb'  # domyślnie 'GeoLite2-Country.mmdb'
GEOIP_CITY = 'Geo_db_city.mmdb'        # domyślnie 'GeoLite2-City.mmdb'
{% endhighlight %}
<p>Pierwsza z nich, GEOIP_PATH jest wymagana. Pozostałe dwie są opcjonalne tak długo, jak trzymamy się domyślnych nazw dla plików z bazami danych miast i państw.</p>
<h3>Pomysł: inteligentny wybór języka i lokalizacji na stronie</h3>
<p>W tej części artykułu wykorzystamy geolokalizację do automatycznego ustawiania języka i lokalizacji na stronie. Oznacza to, że użytkownik odwiedzający naszą witrynę np. z Madrytu, zobaczy ją po raz pierwszy od razu w języku hiszpańskim. Dodatkowo, Hiszpania będzie ustawiona automatycznie jako lokalizacja użytkownika. Jest to nieraz istotne z punktu widzenia biznesu, ponieważ jeśli oferujemy jakieś produkty/usługi na sprzedaż w Internecie, to ich oferta może różnić się pomiędzy krajami. Jak można zaimplementować taką funkcjonalność w Django?</p>
<h3>Przykładowy middleware</h3>
<p>Całkiem sensownymi pomysłami na implementację są dedykowany dekorator funkcji (np. @set_country_and_language) oraz własny middleware (jest to mechanizm typowy dla Django, umożliwia przetworzenie żądania zanim zostanie ono skierowane do obsługi w odpowiednim widoku). W tym przypadku zdecydujemy się na opcję drugą.</p>
<p>Zaczniemy od utworzenia nowego pliku (zwyczajowo zwanego <em>middleware.py</em> w Django) oraz od zdefiniowania w nim kilku potrzebnych importów:</p>
{% highlight python %}
from django.utils import translation
from common.constans import DEFAULT_COUNTRY
from common.geoip.utils import detect_country_by_IP
from common.models import CountryModel
from common.utils import get_language_associated_with_country
{% endhighlight %}
<p>Paczka django.utils.translation odpowiada za włączenie odpowiedniego języka na stronie po wykryciu kraju - i ustaleniu powiązanego z nim języka. DEFAULT_COUNTRY to zwykła stała tekstowa, może być równa np. 'GB'. Oznacza to, że domyślnym krajem będzie Wielka Brytania, jeśli nie zdołamy ustalić go na podstawie adresu IP. detect_country_by_IP to funkcja pomocnicza wykonująca właściwą detekcję kraju. CountryModel to po prostu model kraju w aplikacji. Ostatni import, get_language_associated_with_country, to funkcja pomocnicza, odwzorowująca kraj na język przypisany do niego.</p>
{% highlight python  %}
class GeoIPMiddleware:
    """
    Custom geo location middleware which tries to detect user country and
    language in case of country is not already set in session (like when user
    visits site for the first time).

    Location is set based on request's IP address and by using django.contrib.gis.geoip2 package.
    """
{% endhighlight %}
<p>Middleware jest klasą jak każda inna, nie musi dziedziczyć z żadnej klasy bazowej. Dodajemy zwięzły opis dokumentujący klasę.</p>
<p>Każdy middleware implementuje jedną lub wiele z metod wywoływanych przez Django, najczęściej jest to metoda process_request z żądaniem jako argumentem. Framework wywołuje ją przy każdym żądaniu, jeszcze zanim zdecyduje który widok powinien obsłużyć żądanie:</p>
{% highlight python  %}
def process_request(self, request):
    if 'country' not in request.session:
        detected_country_code = detect_country_by_IP(
            request.META['REMOTE_ADDR'])
        application_country_code = self._get_application_country(
            detected_country_code)
        associated_language = get_language_associated_with_country(
            application_country_code)

        request.session['country'] = application_country_code
        translation.activate(associated_language)

        return None
{% endhighlight %}
<p>Działanie naszego middleware'u polega na obsłużeniu sytuacji, w której klucz 'country' nie jest ustawiony w sesji. W takim przypadku dokonujemy detekcji kraju (za pomocą detect_country_by_IP, adres IP dostępny jest w obiekcie żądania jako request.META['REMOTE_ADDR']). Następnie sprawdzamy czy w bazie danych aplikacji istnieje kraj o podanym kodzie poznanym w wyniku geolokalizacji (metoda _get_application_country, implementacja w dalszej części sekcji) - być może detekcja wskazała na np. Kongo, ale nasza witryna nie obsługuje tego kraju ;) Potem pobieramy język powiązany z krajem (get_language_associated_with_country), ustawiamy kraj w słowniku sesji i aktywujemy odpowiedni język (tłumaczenia) na stronie.</p>
<p>Każdy middleware w Django powinien zwracać&nbsp;None lub obiekt klasy HttpResponse. W naszym przypadku process_request zwraca None, co oznacza, że Django będzie dalej przetwarzać żądanie normalnie.</p>
<h4>Implementacja funkcji pomocniczych</h4>
{% highlight python  %}
def detect_country_by_IP(ip_address):

    """
    Auto-detects user's country based on IP address
    Implementation uses GeoIP2 package

    :param ip_address: IP address as string to look up
    :return: Country code as string
    """

    try:
        geoip = GeoIP2()
        geoip_guess = geoip.country_code(ip_address)
        if geoip_guess is not None:
            detected_country_code = geoip_guess
        else:
            detected_country_code = DEFAULT_COUNTRY
    except GeoIP2Error:
        detected_country_code = DEFAULT_COUNTRY

    return detected_country_code
{% endhighlight %}
<p>Pierwsza z używanych funkcji pomocniczych (detect_country_by_IP) tworzy obiekt klasy GeoIP2 i próbuje pobrać kod kraju na podstawie adresu IP. Jeśli to się udaje, geolokalizacja zakończyła się sukcesem! W przeciwnym wypadku używamy domyślnego kraju.</p>
{% highlight python %}
def _get_application_country(self, proposed_country_code):
    proposed_country_exists = CountryModel.objects.filter(
        country=proposed_country_code).exists()
    if proposed_country_exists:
        return proposed_country_code

    return DEFAULT_COUNTRY
{% endhighlight %}
<p>Następnie wywołujemy metodę _get_application_country. Jeśli aplikacja nie obsługuje kraju o kodzie, jaki wskazała geolokalizacja, używamy domyślnego kraju.</p>
<p>Nie podaję implementacji funkcji get_language_associated_with_country, ponieważ jest ona specyficzna dla danej aplikacji. Każdy kraj może mieć po prostu przypisany do siebie język domyślny, i ta funkcja ma za zadanie ustalenie jaki język powinien zostać ustawiony na stronie dla wykrytego kraju.</p>
<h4>Dodanie middleware do ustawienień aplikacji</h4>
<p>Pozostaje jeszcze dodanie naszego middleware do listy używanych przez aplikację:</p>
{% highlight python %}
MIDDLEWARE_CLASSES = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.locale.LocaleMiddleware',
    ...
    'common.geoip.middleware.GeoIPMiddleware',
]
{% endhighlight %}
<p>Najlepiej zrobić to na końcu listy, ponieważ&nbsp;wówczas będzie on używany po zaaplikowaniu wszystkich poprzednich, pochodzących z frameworka middleware'ów.</p>
<h2>Geolokalizacja - podsumowanie</h2>
<p>Jak starałem się wykazać, temat geolokalizacja nie jest funkcjonalnością ciężką w implementacji, trudną do osiągnięcia. Istnieje gotowa, darmowa biblioteka GeoLite2, za pomocą której możemy dodać geolokalizację do naszej strony w prosty sposób. Dodatkowo, biblioteka ta posiada pełne wsparcie w Django, co jest dodatkowym atutem. Mam nadzieję, że odrobinę rozjaśniłem temat, dzięki za przeczytanie i do następnego wpisu!</p>
