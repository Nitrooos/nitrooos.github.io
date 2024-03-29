---
title: Zarządzanie wydaniami aplikacji - Git workflow
date: 2019-05-08 21:00:05.000000000 +02:00
identifier: managing-releases-git
categories:
- devops
tags:
- deploy
- git
- workflow
permalink: "/2019/05/zarzadzanie-wydaniami-git/"
excerpt: Klient zmienia co chwilę zdanie co powinno zostać wgrane na serwer stagingowy
  albo produkcję? W dzisiejszym wpisie proponuję mechanizm zarządzania wydaniami w
  Gicie, radzący sobie z takimi ciężkimi przypadkami! :)
---
<p>Nieodłącznym elementem naszej ciężkiej, programistycznej pracy jest wgrywanie ukończonych zadań na serwery testowe/przedprodukcyjne, a w końcu na produkcję. Jeśli jesteśmy szczęściarzami, to zadanie po przejściu etapu testów po stronie klienta trafia jak najszybciej na staging a następnie prosto na produkcję. W takim przypadku zarządzanie wydaniami sprowadza się do uaktualniania gałęzi odpowiadających stanom poszczególnych serwerów, np. "develop", "staging" i "production". Niestety, w swojej praktyce często spotykałem sytuację, gdy klient zmieniał zdanie co do kolejności wypuszczania kolejnych zadań. Z dnia na dzień. Z serwera na serwer. Zbiór zadań znajdujących się na serwerze testowym był znacząco różny od zbioru zadań z przedprodukcji. Ten z kolei różnił się od ostatecznej wersji tego, co powinno zostać wgrane na produkcję.</p>
<p>To właśnie z potrzeby bezkonfliktowej współpracy w takich warunkach zrodził się proponowany przeze mnie schemat zarządzania wydaniami. Ma on swoją reprezentację w odpowiedniej organizacji gałęzi w systemie kontroli wersji, w naszym przypadku będzie to git.</p>
<h2>Zarządzanie wydaniami w praktyce</h2>
<p>Dzisiejszy wpis jest zapiskiem zarządzania gałęziami dla standardowego zadania (nazwijmy je "task 1") przy zachowaniu opisanych powyżej założeń:</p>
<ul>
  <li>w każdej chwili chcemy mieć możliwość wgrania <em>dowolnego</em> zadania na serwer testowy (i tylko jego)</li>
  <li>przygotowując wgrywkę (ang. <em>deploy</em>) na serwer przedprodukcyjny chcemy mieć możliwość wyboru dowolnego zbioru zadań przetestowanych na serwerze testowym</li>
  <li>to samo co powyżej dotyczy przygotowania wgrywki na serwer produkcyjny</li>
</ul>
<h3 id="mce_6">Od początku - tworzymy gałąź dla zadania!</h3>
<p>Zakładamy, że główną gałęzią, odzwierciedlającą stan serwera produkcyjnego (ang. <em>live</em>) jest gałąź <em>master</em>. W tym scenariuszu tworzymy nową gałąź dla zadania (nazwijmy ją <em>task_1</em>), wychodzącą z gałęzi <em>master</em>:</p>
<pre>
    $ git checkout master
    $ git pull
    $ git checkout -b task_1
</pre>
<p>Ważne jest, aby uaktualnić lokalny stan gałęzi <em>master</em>, najprościej zrobić to poprzez wydanie polecenia "git pull". Dzięki temu mamy pewność, że pracujemy na aktualnej wersji kodu napędzającej produkcję.</p>
<h3>Praca nad zadaniem</h3>
<p>Od tej pory możemy pracować spokojnie nad zadaniem <em>task_1.</em> W razie potrzeby rozpoczęcia prac nad kolejnym zadaniem powtarzamy procedurę, tworząc nową gałąź i wychodząc ponownie z <em>master</em>'a. Warto w tym miejscu zaznaczyć, że nie należy traktować gita jako narzędzia do archiwizacji wersji kodu "co jakiś czas", "po skończeniu zadania" czy "na koniec dnia". Róbmy commity często, zawierajmy w nich niewielkie, spójne zmiany kodu oraz opatrzmy je krótkim, jasnym komentarzem. Jest to bardzo pomocne w przypadku, gdy zmiany z naszego zadania powodują powstanie błędów regresyjnych. Niewielkie, dobrze opisane commity pomogą odnaleźć dokładną wersję kodu powodującą powstanie błędu. Łatwiejsze będzie także zrozumienie przyczyny jego powstania i jej usunięcie.</p>
<h3>Wgrywka na serwer testowy (opcjonalnie)</h3>
<p>Sprawdzonym przeze mnie schematem jest utrzymywanie serwera testowego, na którym klienci mogą testować nasze zadania. Serwer taki z założenia służy do testowania jednego zadania "na raz". Ma to znaczenie, ponieważ chcemy mieć maksymalną dowolność w doborze zadań, mających być następnie wgranymi na serwer przedprodukcyjny. Nie chcemy po prostu mergować ich do wspólnej gałęzi, tak, aby pozostały niezależne od siebie. Wadą jest oczywiście to, że serwer taki odzwierciedla w danej chwili jedynie stan produkcji + maksymalnie jednego nowego zadania. Z praktyki wiem jednak też, że nie jest to duża wada, szczególnie jeśli skonfigurujemy narzędzie CD (ang. <em>Continous Delivery</em>). Można zrobić to za pomocą np. Jenkinsa, zewnętrznego serwisu (<a href="https://travis-ci.com" target="_blank">Travis CI</a>, <a href="http://codeship.com" target="_blank">Codeship</a>, <a href="https://circleci.com" target="_blank">Circle CI</a>) bądź własnych skryptów, umożliwiając wgrywanie zmian w ciągu kilku minut. Ważne jest, aby <strong>nie mergować takich zadań do żadnej wspólnej gałęzi</strong>. Wgrywanie zmian na serwer testowy powinno odbywać się poprzez wybór gałęzi z zadaniem (czyli np. <em>task_1</em> lub <em>task_2</em>). </p>
<p>Pozostaje kwestia ograniczenia dostępu do takiego serwera, o czym napisałem kilka słów w osobnym <a href="/2019/04/mechanizm-uwierzytelniania-na-serwerach-przedprodukcyjnych/" target="_blank">wpisie</a>.</p>
<h3>Przygotowanie wgrywki na serwer przedprodukcyjny</h3>
<p>Aby przygotować wgrywkę na serwer przedprodukcyjny korzystamy z dedykowanej gałęzi, w naszym przykładzie nazwanej <em>rc_1</em>. Takich gałęzi, służących do wgrywania zmian na serwer przedprodukcyjny może być więcej w razie potrzeby. Osobiście, standardowo nazywam je <em>rc_2</em>, <em>rc_3</em> itd. Są one przydatne w sytuacji, gdy na staging wgrano np. zmiany składające się na zadania 1, 2 i 3, po czym klient chce wgrać na produkcję tylko zadania 1 i 3. W takim przypadku tworzymy np. na gałęzi <em>rc_2</em> (zawsze wychodzącej z aktualnego stanu gałęzi podstawowej, <em>master</em>!) nowy release, składający się tylko z zadań 1 i 3. Więcej o takim przypadku napisałem w kolejnym punkcie tego wpisu.</p>
<p>Przygotowanie wgrywki na serwer przedprodukcyjny sprowadza się do zmergowania zadań, które mają się w nim znaleźć do odpowiedniej gałęzi (najczęściej <em>rc_1</em>). Wgrywany na serwer jest aktualny stan wybranej gałęzi <em>rc_*</em>. Na tym etapie możemy dobrać dowolne spośród gotowych zadań.</p>
<h3>Testowanie na serwerze przedprodukcyjnym</h3>
<p>Wgrywanie zmian na produkcję musi być poprzedzone przetestowaniem ich na serwerze przedprodukcyjnym. Najlepiej aby zajął się tym profesjonalny tester manualny. Świetnie sprawdzają się napisane specjalnie do tego celu testy automatyczne (end-to-end). Do ich stworzenia polecić mogę np. narzędzie <a href="https://www.cypress.io/" target="_blank">cypress</a>.</p>
<h4>Klient zmienił zdanie! Mamy wgrać na produkcję inny zbiór zadań...</h4>
<p>Czasami zdarza się, że priorytet jednego z zadań/część zadań wgranych na serwer przedprodukcyjny ulega zwiększeniu. Powinniśmy w takim wypadku wgrać je na produkcję <em>jak najszybciej</em>. Dla przykładu, wgrano na serwer przedprodukcyjny zadania 1, 2 oraz 3. Teraz jednak na produkcję powinny zostać wgrane tylko zadania 1 i 3. Prawidłowym postępowaniem w takiej sytuacji jest przygotowanie nowego release'u np. na gałęzi <em>rc_2</em>, zawierającego tylko zadania 1 i 3 (uzyskały większy priorytet), dokonanie wgrywki na serwer przedprodukcyjny z gałęzi <em>rc_2</em> i testowanie. </p>
<p>Dlaczegóżby jednak nie wgrać od razu na produkcję zadań 1 i 3, szczególnie jeśli zostały już przetestowane? </p>
<p>Po pierwsze, nie chcemy przenosić na gałąź <em>master</em> zmian ręcznie, commit po commicie wybranych zadań. Musielibyśmy to zrobić, ponieważ gałąź <em>rc_1</em> zawiera zmiany także z zadania 2, które jednak nie powinno znaleźć się teraz na produkcji. Można to zrobić np. za pomocą komendy <em>git cherry-pick,</em> jest to jednak sposób długotrwały i podatny na błędy. </p>
<p>Po drugie, zadania 1 i 3 mogą być zależne zadania 2, które teraz należy pominąć. Możliwa jest więc sytuacja, w której wgrane razem zadania 1, 2 i 3 działają bez zarzutu, jednak bez zadania 2. pojawiają się błędy. Ważne jest więc, aby na produkcję <strong>wgrywać tylko stan kodu, który był już wcześniej sprawdzony</strong> na serwerze przedprodukcyjnym. Wgranie tylko zadań 1 i 3 w naszym przykładzie byłoby błędem. Jest tak dlatego, że sprawdzone zostało tylko działanie ich razem z zadaniem 2. Nie oznacza to automatycznie, że również bez niego wszystko będzie działać poprawnie.</p>
<h3>Wgrywka na produkcję</h3>
<p>Ostatnim etapem jest wgranie zmian na produkcję, którego dokonujemy na żądanie klienta oraz, powtórzę raz jeszcze, <strong>tylko po wcześniejszym przetestowaniu identycznej wersji kodu na serwerze przedprodukcyjnym</strong>. Rozpoczynamy oczywiście od zmergowania gałęzi zawierającej wersję kodu testowaną na serwerze przedprodukcyjnym (np. <em>rc_1</em>) do gałęzi <em>master</em>. Następnie proponuję tagowanie wersji na gałęzi <em>master</em>, tak, aby kolejne release'y kodu były uporządkowane, ponumerowane i odpowiednio opisane. Do stworzenia tagów w Gicie służy komenda:</p>
<pre>
    $ git tag -a v0.1 -m 'Minimal Valueable Product'
</pre>
<p>Natomiast aby wgrać je do zdalnego repozytorium możemy posłużyć się komendą:</p>
<pre>
    $ git push origin v0.1
</pre>
<p>Narzędzie służące do przygotowania wgrywki na produkcję powinno akceptować nazwę tagu jako parametr. Dzięki temu wgrywane będą tylko przygotowane w tym celu, odpowiednie wersje kodu.</p>
<h2>Zarządzanie wydaniami na konkretnym przykładzie</h2>
<p>Pokażę jeszcze przykład zarządzania wydaniami na przykładzie 4 zadań, 2 wgrywek na serwer przedprodukcyjny i 2 wgrywek na produkcję. Sytuację przedstawia poniższy schemat:</p>
![
  Zarządzanie wydaniami w systemie kontroli wersji - historia prac nad zadaniami
]({{ site.baseurl }}/assets/img/2019-05-08/workflow-384x1024.png)
*Przykładowa historia prac nad zadaniami i zarządzania ich gałęziami*

<p>Gałęzie <em>rc_1</em> i <em>rc_2</em> służą do przygotowania wgrywek na przedprodukcyjny serwer, gałąź <em>master</em> odzwierciedla stan kodu na produkcji, gałęzie z zadaniami nazywają się <em>task_1</em>, <em>task_2</em>, <em>task_3</em> i <em>task_4</em>. Linia przerywana czerwona oznacza okres testowania zadania na serwerze testowym. Linia przerywana szara oznacza, że zadanie jest gotowe, ale aktualnie nie jest wgrane na serwer testowy. Jak pamiętamy, może na nim być jednocześnie wgrane tylko 1 zadanie. </p>
<h3>Analiza krok po kroku</h3>
<p>Rozpoczęto pracę nad zadaniem <em>task_1</em> (wychodząc nową gałęzią z <em>master</em>'a), ukończono pracę i wgrano na serwer testowy. W międzyczasie rozpoczęto pracę nad kolejnym zadaniem (<em>task_2</em>). Po ukończeniu pracy zostało ono wgrane na serwer testowy (zamiast zadania <em>task_1</em> - linia przerywana szara). W trakcie gdy na serwerze testowym znajdowało się zadanie <em>task_2</em>, rozpoczęto pracę nad priorytetowym zadaniem <em>task_3 (hotfix)</em>. Zadania <em>task_1</em> i <em>task_2</em> zostały zmergowane do gałęzi <em>rc_1</em>. Po zakończeniu prac nad zadaniem <em>task_3</em> i przetestowaniu go na serwerze testowym zmergowano go do gałęzi <em>rc_2</em>. Zrobiono tak, ponieważ zadanie to powinno znaleźć się na produkcji jak najszybciej, bez czekania na przetestowanie i ewentualne poprawki do zadań <em>task_1</em> i <em>task_2</em>. Z gałęzi <em>rc_2</em> wgrano kod na serwer przedprodukcyjny, przetestowano go i zmergowano do gałęzi <em>master</em>. Następnie stworzono nowy tag <em>v0.1</em> i wgrano kod na produkcję.</p>
<p>Dalej, zmiany z gałęzi <em>master</em> zostały włączone do gałęzi <em>rc_1</em> z zadaniami <em>task_1</em> i <em>task_2</em> (ważne!). Po przetestowaniu, zmergowano gałąź <em>rc_1</em> do <em>master</em>, stworzono nowy tag <em>v0.2</em> i wgrano kod na produkcję.</p>
<p>Na dzisiaj to już wszystko, mam nadzieję, że przedstawiłem jasno proponowany przeze mnie schemat zarządzania wydaniami aplikacji w praktyce. Oraz że komuś z Was się on przyda w codziennej pracy. A może macie lepsze/inne sprawdzone schematy zarządzania gałęziami i wgrywkami? Zapraszam do dyskusji w komentarzach!</p>
