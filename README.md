# Estimation-of-apartment-price

В этом проекте будет реализована идея предсказания цены квартиры по некоторым признакам. Хоть тема довольно и популярная, однако позволяет затронуть и углубится довольно сильно во все стадии разработки ML проекта.

## Парсинг данных с авито 

Данные парсил с авито, так как на этом сайте минимальное присуствие js, из-за чего можно доставать информацию не используя сторонних приложений, пользуясь лишь библиотеками python парсинга BeautifulSoup, urrlib, requests.

Так как Авито активно борется с автоматическим парсингом, то даже при найденных 50 тыс. объявлений, доступно лишь около 4 тыс. Поэтому приходилось задавать разные параметры в фильтрах поиска объявлений, чтобы получать максимально разные выборки объявлений. Таким образом удалось собрать около 42 тыс. объявлений. Все равно мы не застрахованы от дубликатов объявлений, поэтому в будущем учтем это.

## Предобработка данных 

Чистка данных от выбросов, обработка nan значений и ошибок парсинга.

В процессе парсинга данные парсились неточно, в значения признаков попадали лишние символы или наоборот. В данном разделе я это исправил.

Также в процессе парсинга некоторые значения признаков для некоторых объектов не удалось найти, поэтому в нашем наборе данных появились nan значения. Я пытался восстановить неоторые nan значения, где это было возможно. Если восстановить данные было невозможно, то я не удалял эти значения, а присваивал им значения меньше минимального значения соответсвующего признака. Тем самым, градиентный бустинг будет отправлять эти значения всегда в один лист.

Также я столкнулся с выбросами в датафрейме, которые удалял в большей мере ориентируясь на диаграммы размахов (boxplot).

Работа по добыче информации из сырых признаков.

После парсинга мы встретились с большим количеством сырых признаков, например, координаты, название метро. Их необходимо было преобразовать во что-то разумное. Например, координаты можно преобразовать в расстояние до центра и в район, а метро в ветку и удаленность станции от центральных станций.

Кодировка признаков.

Я использовал для кодировки Mean Target Encoding и Label Encoding. MTE, как показали тесты, довольно хорошо показывает себя на xgboost и lightgbm. LE кодировку мы оставим для catbosst, т.к. catboost довольно хорошо обрабатывает категориальные признаки сам, насколько мне известно, он также использует target encoding.

## Создание и обработка взаимодействий между признаками

В нашем наборе данных есть признаки, которые скоррелированы между собой. Это прежде всего связано с тем, что значения одного признака могут вытекать из значений другого признака. Эту зависимось можно постараться убрать вычитанием одного признака из другого или делением, где это имеет смысл. Проверяем новые значения признаков и если они дают лучше качество, то оставляем эти изменения признаков.

Для увеличения гибкости нашей модели можно добавить взаимодействия между признаками. Если эти признаки будут излишними, то мы удалим их в процессе отбора признаков. Поэтому добавим взаимодействия между признаками прежде всего пользуясь логикой, какие взаимодействия действительно имеют смысл. Также добавим взаимодействия между признаками, которые сильно влияют на цену, например, район и общая площадь или расстояние до Кремля и общая площадь.

## Отбор признаков

Будем использовать несколько методик для отбора признаков. 

Split. Основывается на количестве применений конкретного признака для разделения узла за все время обучения.

Permutation importance. Модель делает предсказание на каком то блоке данных, получает какое-то качество. Потом конкретный признак перемешивается, предсказание делается вновь и мы получаем новое качество. Важность признака основывается на разности качество до и после перемешивания.

Жадное удаление признаков. Признаки сначала сортируются какими-то другим методом, например, split, от менее важного к более важному. Начиная с менее важных признаков, мы удаляем признак и сравниваем качество до удаления и после, если качество возросло, то мы удаляем признак, если нет- оставляем, и переходим к следующему признаку, качество которого мы смотрим уже на обновленном наборе данных.

Split в динамике. Мы смотрим split в зависимости от глубины дерева. Тем самым можем смотреть на какой стадии начинают применяться те или иные признаки. Те признаки, которые применяются на маленькой глубине, являются крайне важными, потому что они применяются в первую очередь.

Модифицированный backward elimination features.

Суть обычного backward elimination features заключается в том, что мы на каждой итерации алгоритма удаляем признак с самым большим p-значением и делаем это до тех пор, пока качество модели не достигнет пика или пока не будут удалены все признаки. Мы же вместо p-значения будем использовать разность в качестве, удаляя признак с самым минимальным приростом качества. Мы пройдемся по удалению всех признаков и получил ранжированный по важности список признаков.

В итоге мы удалили лишь следующие признаки: .

## Важность и влияние признаков

Будем использовать shap.

Основа этого метода заключается в вычислении shap values для каждого признака для конкретного объекта. shap_values вычисляется следующим образом: мы создаем коалиции с разным количеством включенных в коалицию признаков от нуля до всего количества признаков. И смотрим как меняется предсказание на объекте на конкретной коалиции при включении и исключении конкретного признака. Эти изменения для конкретных коалиций аггрегируются с какими-то весами. То есть если shap_value положительное, это значит, что включение признака повышает в среднем предсказание на конкретном объекте.

## Финальная стек-модель

Будем использовать стек-модель. Стекат будем xgboost, catboost, lightgbm. 
Будем использовать следующую стратегию:

1. 10% отложенная выборка.
2. На оставшихся данных обучаем три модели, используя кросс-валидацию. Предсказываем на не обучающем фолде значения тремя моделями.
3. На тех же индексах, что и обучались базовые модели, мы обучаем стек-модель и предсказываем на не обучающих фолдах, получая какое-то качество. Обучаем мы на тех же индексах, чтобы избежать утечки данных. То есть таким образом мы избегаем излишнего переобучения. В качестве подтверждения, что если не использовать одни и те же индексы для обучения, то у нас будет утечка данных, прикрепляю это обсуждению на кэгле, где пользователь Faron подтвердил это: https://www.kaggle.com/general/18793
4. На отложенной выборке проверяем итоговое качество нашей модели. Подгружаем все наши модели, делаем предсказание на отложенной выборке, эти предсказания используем для предсказания нашей стек-модели. MAPE в результате получился 

## Предсказание на данных от пользователя

В качестве входных параметров от пользователя будем использовать: адрес, метро, время до метро, количество этажей в доме, номер этажа, площадь, площадь кухни, год постройки дома, количество комнат, ремонт.
Адрес мы преобразуем в расстоянии до Кремля и в район. Метро мы преобразуем в ветку и расстояние до центральной станции.Также преобразуем и создадим взаимодействия между признаками. Сделаем предсказание.
