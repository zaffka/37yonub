---
layout: post
title:  "Январь 2018, что с прогрессом"
date:   2018-02-01
categories: прогресс
comments: true
---
Январь-месяц оцениваю как удавшийся. Есть успехи, есть и проблемы. Но о последних в конце.

**Главное** - декабрьский откат, о котором я писал в предыдущем отчёте, успешно преодолён.
**Осознание месяца** - курсы больше не нужны, уровень знаний уже таков, что имеющиеся на Pluralsight и других ресурсах материалы ничего нового мне не открывают. Вместо курсов читаю документацию на те или иные необходимые модули, смотрю примеры кода. И это уже, в общем, соответствует нормальному процессу разработки.

## Pet-project
**Бэк**.
В процессе возвращения "в струю" завершил API домашнего проекта (флэшкарточки для заучивания иностранных слов). Кому интересно, вот ссылка на [репозиторий API на github](https://github.com/zaffka/newwords){:target="_blank"}, рядышком [лежит webapp](https://github.com/zaffka/newwords-web){:target="_blank"}. На следующей неделю докручу генерируемую API-документацию, заполню README-шки.
Если успею, подниму docker-а на Digitalocean и разверну сервис в контейнерах.

**Что сейчас умеет API**.
Читать, создавать, обновлять, удалять отдельные слова(флэшкарточки). Читать и отдавать список флэшкарточек. Создавать новых пользователей, аутентифицировать пользователей, авторизовывать их запросы используя jwt-токен. Отправлять письма, активировать пользователя, сбрасывать пароль, перезапрашивать активацию. База - Postgres. Приложение запаковано в докер-контейнер, настройки вынесены в переменные окружения.

**Фронт**.
Последние пару недель процентов на 90% ушли как раз на веб-приложение. Строю его на связке Vue.js + Axios + Bootstrap + Animate.css

Вот что получилось:
![Изучение слов](/assets/img/Peek2018-02-01.gif){:class="img-responsive"}

В принципе, тем, что вышло, я доволен, хотя и не сомневаюсь, что внутри JS и вёрстки изрядный говнокод. Но необходимый минимум есть. Пока на этом закончу, надо вернуться к Go, иначе получится слишком большой перерыв. Тут важно соблюсти баланс и не получить откат по Go ещё раз.

**Есть проблема** на участке баз данных.

Пришло понимание, что полноценно впихнуть в свой текущий график изучение SQL не могу. Понятно, несложные запросы я и до Go умел строить, но этого не достаточно, нужно серьёзно добирать знаний. А раз времени на изучение SQL нет, придётся пока подрезать планы.

В ближайшее время переведу API проекта с Postgres-а на BoltDB. Структура данных у меня простенькая, Bolt-а будет достаточно.

Но знание хотя бы одной базы - это обязательное условие, которое я сам для себя определил, как отправную точку для начала поиска работы.
По совету старшего товарища, попробую закрыть эту прореху NoSQL-решением. Конкретно - монгой (MongoDB). Адских join-ов там нет, худо-бедно знакомый js-синтаксис. Товарищ обещал, что будет попроще ;)

Кроме того, в версии 3.6 mongodb, говорят, появились change streams. Это интересно, это может пригодиться. Беру! :))
Следующий major-курс будет по монге. 
