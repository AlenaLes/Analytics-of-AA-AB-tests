# Анализ результатов АА и АВ тестирования

## Краткое описание

Анализируем данные по приложению, объединяющего ленту новостей и ленту сообщений. Для этого используем результаты АА и АВ тестов, которые были проведены ранее.

Стэк:

- JupiterHub
- Clickhouse
- Python
- Статистические тесты

## АА - тест
Убедимся, что система сплитования работает корректно и ключевая метрика не отличается между группами в АА-тесте.

Данные выгрузили с Clickhouse, всего у нас 42 585 пользователй для АА-теста. Также поставили ограничение по времени выгрузки: с 26.09.2022 по 2.10.2022 и выбрали только 2-ую (экспериментальную) и 3-ью (экспериментальную) группы.

Для проведения теста подсчитали CTR для каждого пользователя и построили график с распределением для информации. На графике видим, что распределение экспериментальных групп практически соврадает.

![image](https://user-images.githubusercontent.com/100629361/206895677-7b6cfc46-f868-4afd-af04-186b8a86284f.png)

Теперь для проверки различий проведем бутстрап.

```
# Проведем бутстрап

def bootstrap_aa(n_bootstrap: int = 10_000, 
                 B: int = 500,
) -> List:
    pvalues = []
    for _ in range(n_bootstrap):
        group_2 = df_aa[df_aa['exp_group'] == 2]['ctr'].sample(B, replace=False).tolist()
        group_3 = df_aa[df_aa['exp_group'] == 3]['ctr'].sample(B, replace=False).tolist()
        pvalues.append(st.ttest_ind(group_2, group_3, equal_var=False)[1])
    
    return pvalues
```

```
perc = sum([1 for i in pvalues if i <= 0.05]) / 10000 * 100
print(f'{perc}% is the percent of p-values which are less than 0.05 ')

4.569999999999999% is the percent of p-values which are less than 0.05 
```

Статистически значимые различия получаем в 4,6% , что соответствует ошибке I рода. 
Система сплитования на группы работает корректно.

## АВ - тест

Также, как и в АА-тесте выгрузили данные с Ckickhouse. Ограничение по времени выгрузки: с 3.10.2022 по 9.10.2022 и выбрали только 1-ую (контрольную) и 2-ую (экспериментальную) группы.

Различия распределений по CTR очень хорошо прослеживается. T-test не даст нам гарантированного результата, поэтому будет использовать разные тесты для оценки.

![image](https://user-images.githubusercontent.com/100629361/206896119-ba713976-9dab-4059-805d-6feab1a84784.png)

## T-test

P-value  больше 0,05, поэтому статистически значимого различия нет. Это зависит от того, что распределение ctr второй группы не нормальное, применение теста не корректно.

```
st.ttest_ind(group_a, group_b, equal_var=False)

Ttest_indResult(statistic=0.7094392041270486, pvalue=0.4780623130874935)
```
```
 group_a.mean(), group_b.mean()
 
(0.21560459841296287, 0.21441927347479375)
```
## Mann-Witney test

Тест Манна Уитни показывает статистические различия.

```
st.mannwhitneyu(group_a, group_b, alternative='two-sided')

MannwhitneyuResult(statistic=56601260.5, pvalue=6.0376484617779035e-56)
```

## Сглаженный CTR

Применим сглаживание для повтрного проведения тестов.
![image](https://user-images.githubusercontent.com/100629361/206896354-2a4b42f6-1951-4e60-bdee-bafd72fa5060.png)

## Повторно проводим тесты на сглаженный CTR

Сглаженный ctr T-test, в отличие от обычного ctr, показывает статистически значимое различие. Тест Манна-Уитни сходится с предыдущим.

![image](https://user-images.githubusercontent.com/100629361/206896427-e05650d1-8baf-44f8-9d0e-02847dba37c4.png)

## Poisson bootstrap

Распределение разницы глобального ctr показывает, что ctr в тестовой группе стал меньше для каждой бустрап-подвыборки
![image](https://user-images.githubusercontent.com/100629361/206896519-cd46c076-2514-4e2a-a993-882bf01abfc6.png)

## Mann-Witney test

Для проведения теста Манна-Уитни сначала провели бакетное преобразование. Аидим, что в среднем глобальный CTR уменьшился.

![image](https://user-images.githubusercontent.com/100629361/206896802-38fb13de-26a3-47e6-916b-d1f2bd5fda41.png)

По результатам теста не рекомендуется раскатывать новый алгоритм на всех пользователей.
