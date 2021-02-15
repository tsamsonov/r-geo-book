# Географически взвешенная регрессия {#gwr}




```r
library(sf)
library(RColorBrewer)
library(readxl)
library(tidyverse)
library(tmap)
library(GWmodel)
```

[Программный код главы](https://github.com/tsamsonov/r-geo-course/blob/master/code/15-SpatstatAutocorrelation.R)

## Географически взвешенная регрессия (GWR) {#autocorrelation_gwr}

В стандартной модели линейной регрессии параметры модели предполагаются постоянными:

$$\mathbf{y} = \mathbf{X} \boldsymbol\beta + \boldsymbol\epsilon,$$

Для $i$-й локации решению выглядит следующим образом:

$$y_i = \beta_0 + \beta_1 x_{1i} + \beta_2 x_{2i} + ... + \beta_m x_{mi} + \epsilon_i$$

Коэффициенты находятся методом наименьших квадратов:

$$\mathbf{\beta}' = (\mathbf{X}^T\mathbf{X})^{-1} \mathbf{X}^T \mathbf{Y}$$

Такой подход, однако не учитывает того, что характер зависимости между переменными может меняться по пространству.

В __географически взвешенной регрессионной модели__ веса определяются для каждой локации:

$$y_i = \beta_{0i} + \beta_{1i} x_{1i} + \beta_{2i} x_{2i} + ... + \beta_{mi} x_{mi} + \epsilon_i$$

В этом случае область оценки параметров $\mathbf{\beta}$ ограничивается некой окрестностью точки $i$. Математически это достигается применением весовых коэффициентов для данных независимых переменных:

$$\mathbf{\beta}'(i) = (\mathbf{X}^T \color{blue}{\mathbf{W}(i)}\mathbf{X})^{-1} \mathbf{X}^T \color{blue}{\mathbf{W}(i)} \mathbf{Y},$$

где $\mathbf{W}(i)$ есть матрица весов для точки $i$. Коэффициенты матрицы подбираются таким образом, что близкие локации получают более высокий вес.

Матрица $\mathbf{W}(i)$ имеет размер $n \times n$, где $n$ — число точек наблюдений:

$$\mathbf{W}(i) = \begin{bmatrix}
    w_{i1} & 0 & 0 & \dots  & 0 \\
    0 & w_{i2} & 0 & \dots  & 0 \\
    0 & 0 & w_{i3} & \dots  & 0 \\
    \vdots & \vdots & \vdots & \ddots & \vdots \\
    0 & 0 & 0 & \dots  & w_{in}
\end{bmatrix},$$

где $w_{ik}$ есть вес, который имеет точка $k$ при локальной оценке параметров в точке $i$.

### Весовые функции

Весовая функция должна быть убывающей. Существует множество вариантов таких функций, но наиболее часто используются гауссоподобные варианты:

<div class="figure">
<img src="slides/images/gwr_wlocal.png" alt="Весовая функция" width="100%" />
<p class="caption">(\#fig:unnamed-chunk-2)Весовая функция</p>
</div>

В случае _фиксированной_ весовой функции окрестность всегда имеет фиксированный размер:

$$w_{ij} = \operatorname{exp}\{-\frac{1}{2} (d_{ij}/h)^2\},$$

где $d_{ij}$ есть расстояние, $h$ — полоса пропускания.

<div class="figure">
<img src="slides/images/wfixed.png" alt="Фиксированная весовая функция" width="100%" />
<p class="caption">(\#fig:unnamed-chunk-3)Фиксированная весовая функция</p>
</div>

В случае _адаптивной_ весовой функции окрестность ограничивается $N$ ближайшими точками. За пределами этой окрестности веса принимаются равными нулю:

<div class="figure">
<img src="slides/images/wadaptive.png" alt="Адаптивная весовая функция" width="100%" />
<p class="caption">(\#fig:unnamed-chunk-4)Адаптивная весовая функция</p>
</div>

__Полоса пропускания__ $h$ обладает следующими особенностями:

- малая полоса пропускания приводит к большой дисперсии локальных оценок;
- большая полоса пропускания приводит к смещенности оценки;
- при $h \rightarrow \infty$ локальная модель приближается к _глобальной регрессии_;
- при $h \rightarrow 0$ локальная модель «сворачивается» вокруг данных.

### Практический анализ

В качестве примера проанализируем каким образом цена жилья зависит от количества комнат на примере данных по стоимости недвижимости в Бостоне, доступных на [данном сайте](https://www.jefftk.com/apartment_prices/data-listing), и выгруженных с североамериканского информационного портала недвижимости [padmapper.com](https://www.padmapper.com/apartments/boston-ma?box=-71.2297689,42.2745193,-70.9215169,42.4531515):


```r
realest = read_delim(url('https://www.jefftk.com/apartment_prices/apts-1542637382.txt'),
                 delim = ' ',
                 col_names = c('price', 'rooms', 'id', 'lon', 'lat')) %>%
  st_as_sf(coords = c('lon', 'lat'), crs = 4326) %>%
  st_transform(3395)

# tmap_mode('view')
tm_shape(realest) +
  tm_bubbles(col = 'price',
             size = 'rooms',
             style = 'fixed',
             breaks = c(0, 1000, 2000, 3000, 4000, 5000, 10000, max(realest$price)),
             scale = 0.25,
             palette = colorRampPalette(c('steelblue4', 'orange', 'darkred'))(7),
             alpha = 0.8) +
  tm_view(symbol.size.fixed = TRUE)
```

<img src="19-GWRegression_files/figure-html/unnamed-chunk-5-1.png" width="100%" />

Для того чтобы оценить пространственую неравномерность реакции стоимости жилья на увеличение количества комнат, построим модель географически взвешенной регрессии:

```r
samples = realest %>% dplyr::sample_n(1000) %>% as('Spatial')

(gwr_res = gwr.basic(price ~ rooms, data = samples, bw = 1000, kernel = 'gaussian'))
##    ***********************************************************************
##    *                       Package   GWmodel                             *
##    ***********************************************************************
##    Program starts at: 2021-02-15 20:22:55 
##    Call:
##    gwr.basic(formula = price ~ rooms, data = samples, bw = 1000, 
##     kernel = "gaussian")
## 
##    Dependent (y) variable:  price
##    Independent variables:  rooms
##    Number of data points: 1000
##    ***********************************************************************
##    *                    Results of Global Regression                     *
##    ***********************************************************************
## 
##    Call:
##     lm(formula = formula, data = data)
## 
##    Residuals:
##     Min      1Q  Median      3Q     Max 
## -1820.7  -754.1  -320.8   279.2 13475.3 
## 
##    Coefficients:
##                Estimate Std. Error t value Pr(>|t|)    
##    (Intercept)  2287.32      79.36   28.82   <2e-16 ***
##    rooms         433.38      31.99   13.55   <2e-16 ***
## 
##    ---Significance stars
##    Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1 
##    Residual standard error: 1337 on 998 degrees of freedom
##    Multiple R-squared: 0.1553
##    Adjusted R-squared: 0.1545 
##    F-statistic: 183.5 on 1 and 998 DF,  p-value: < 2.2e-16 
##    ***Extra Diagnostic information
##    Residual sum of squares: 1784839706
##    Sigma(hat): 1337.317
##    AIC:  17238.72
##    AICc:  17238.74
##    ***********************************************************************
##    *          Results of Geographically Weighted Regression              *
##    ***********************************************************************
## 
##    *********************Model calibration information*********************
##    Kernel function: gaussian 
##    Fixed bandwidth: 1000 
##    Regression points: the same locations as observations are used.
##    Distance metric: Euclidean distance metric is used.
## 
##    ****************Summary of GWR coefficient estimates:******************
##                 Min. 1st Qu.  Median 3rd Qu.   Max.
##    Intercept  151.58 1791.09 1998.30 2323.63 4253.5
##    rooms     -210.12  448.84  539.34  695.19 1428.0
##    ************************Diagnostic information*************************
##    Number of data points: 1000 
##    Effective number of parameters (2trace(S) - trace(S'S)): 116.9521 
##    Effective degrees of freedom (n-2trace(S) + trace(S'S)): 883.0479 
##    AICc (GWR book, Fotheringham, et al. 2002, p. 61, eq 2.33): 17063.91 
##    AIC (GWR book, Fotheringham, et al. 2002,GWR p. 96, eq. 4.22): 16956.18 
##    Residual sum of squares: 1239492307 
##    R-square value:  0.4133962 
##    Adjusted R-square value:  0.3356175 
## 
##    ***********************************************************************
##    Program stops at: 2021-02-15 20:22:55

tm_shape(gwr_res$SDF) +
  tm_bubbles(col = 'rooms', # это не количество комнат, а коэффициент регрессии
             style = 'quantile',
             scale = 0.3,
             palette = 'Reds',
             alpha = 0.5) +
  tm_view(symbol.size.fixed = TRUE)
```

<img src="19-GWRegression_files/figure-html/unnamed-chunk-6-1.png" width="100%" />

Как видно, модель GWR наглядно показывает наличие пространственной гетерогенности (неоднороности) в распределении показателя. Четко видны районы (в основном цеентральные, но также и часть окраинных), где стоимость жилья резко возрастает при увеличении количества комнат.

## Краткий обзор {#gwr_review}

Для просмотра презентации щелкните на ней один раз левой кнопкой мыши и листайте, используя кнопки на клавиатуре:
<iframe src="https://tsamsonov.github.io/r-geo-course/slides/15-SpatialRegression_slides.html#1" width="100%" height="500px"></iframe>

> Презентацию можно открыть в отдельном окне или вкладке браузере. Для этого щелкните по ней правой кнопкой мыши и выберите соответствующую команду.

## Контрольные вопросы и упражнения {#questions_tasks_gwr}

### Вопросы {#questions_gwr}

1. Сформулируйте задачу и напишите уравнение географически взвешенной регрессии. Чем обусловлена необходимость в использовании такой модели?
1. Определите назначение весовой функции в методе географически взвешенной регрессии. Каков ее основной параметр?

### Упражнения {#tasks_gwr}


----
_Самсонов Т.Е._ **Визуализация и анализ географических данных на языке R.** М.: Географический факультет МГУ, 2021. DOI: [10.5281/zenodo.901911](https://doi.org/10.5281/zenodo.901911)
----