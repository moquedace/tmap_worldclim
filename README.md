---
Título: "Tutorial - Mapas arranjados com`tmap`"
Autores: "Cássio Moquedace, Khaterine Martinez, Lara Lima, Rugana Imbana"
---

## Objetivo
Criar mapas arranjados com o `tmap` a partir de dados históricos e modelos futuros de precipitação

<p>&nbsp;</p>

## WorldClim
As mudanças climáticas nas últimas décadas tiveram impacto generalizado nos sistemas naturais e humanos, observáveis em todos os continentes. Modelos ecológicos e ambientais que usam dados climáticos geralmente dependem de dados em grade, como o WorldClim.

O WorldClim é um banco de dados bioclimáticos globais, de alta resolução espacial. Esses dados contêm também modelos e cenários, que podem ser usados para mapeamento e modelagem espacial. Nesse contexto, para construção desse tutorial serão utilizados dados climáticos históricos (1970 - 2000) da versão 2.1 e as predições climáticas futuras do modelo `MIROC6` para o cenário *“Shared Socioeconomic Pathways”* 8.5 (ssp585).

<p>&nbsp;</p>

## `tmap`
Aprenderemos como plotar dados espaciais em R usando o pacote `tmap`. Este pacote detém uma gramática de gráficos para elaboração de mapas temáticos que se assemelha à sintaxe do `ggplot2`. Este pacote é poderoso e útil para exploração e publicação de dados espaciais, além de oferecer plotagem estática e interativa.

<p>&nbsp;</p>

## Execução dos códigos
### Carregando pacotes necessários e definindo local de trabalho
Limpando memória não utilizada no R
```{r message=FALSE}
gc()
```

Carregando pacotes
```{r message=FALSE}
pkg <- c("dplyr", "tmap", "tmaptools", "sf", "geobr", "raster", "rgdal", "spData", "spDataLarge", "stringr")

sapply(pkg, require, character.only = T)
```

Limpando todos os objetos gravados no R
```{r message=F}
rm(list = ls())
```

Definindo local de trabalho
```{r message=FALSE, eval=FALSE, echo=TRUE}
setwd("D:/OneDrive/Cássio/R/sol 793/tuto_2")
```

<p>&nbsp;</p>

### Baixando dados 1970 - 2000 WorldClim
Para baixar dados do período 1970 - 2000 utilizaremos a função `getData` do pacote `raster`, para detalhes a respeito de outras variáveis disponíveis no WorldClim para download via `getData` utilize o `help` da função `?getData`.

Maiores detalhes dos dados diponíveis no `WorldClim` acesse [WorldClim](https://www.worldclim.org/)

Baixando dados 1970 - 2000
```{r message=FALSE}
bioclim <- getData(download = T, name = "worldclim", var = "bio", res = 2.5,
                   path = "../dados/atual/bioclim")
```

<p>&nbsp;</p>

### Preparando dados
Verificando nome das camadas empilhadas das 19 bioclimáticas
```{r message=FALSE}
names(bioclim)
```

Utilizaremos a bioclimática 12 (`BIO12 = Annual Precipitation`). Os detalhes do significado de cada bioclimática no site [WorldClim](https://www.worldclim.org/data/bioclim.html)

Separando a camada com a Precipitação Anual Média 1970 - 2000
```{r message=FALSE}
presente <- bioclim[["bio12"]]
```

#### Obtenção de dados futuros
A função `getData` do pacote `raster` não fornece suporte para download de dados futuros da versão 2.1 do WorldClim lançada no início do ano de 2020.

Podemos baixar diretamente no site do `WorldClim` [clicando aqui](http://biogeo.ucdavis.edu/data/worldclim/v2.1/fut/2.5m/wc2.1_2.5m_bioc_MIROC6_ssp585_2041-2060.zip)

Carregando Precipitação Mensal Média futura (2041 - 2060) e multiplicando por 12 para converter em Precipitação Anual Média (PAM)
```{r message=FALSE}
futuro <- raster("../dados/futuro/ssp585/wc2.1_2.5m_prec_MIROC6_ssp585_2041-2060.tif") * 12
```

#### Visualizando dados 1970 - 2000 e futuro
```{r message=FALSE, fig.width=10, fig.height=5}
plot(presente)
plot(futuro)
```

<p>&nbsp;</p>

### Recortando continente continete africano
Carregando `shapefile` do mundo, filtrando continente africano, alterando a quantidade de `character` para quebra de texto no nome dos países e reprojetando para o datum das imagens WorldClim
```{r message=FALSE}
africa <- world %>% 
  filter(continent == "Africa") %>% 
  mutate(name_long2 = str_wrap(name_long, 10)) %>% 
  st_transform(crs = st_crs(futuro))
```

Visualizando `shapefile`
```{r message=FALSE, fig.width=10, fig.height=4}
plot(africa$geom)
```

Recortando imagem 1970 - 2000
```{r message=FALSE}
mask_presente <- (presente) %>% 
  crop(africa) %>% 
  mask(africa)
```

Recortando imagem futuro
```{r message=FALSE}
mask_futuro <- (futuro) %>% 
  crop(africa) %>% 
  mask(africa)
```

<p>&nbsp;</p>

### Criando mapa
Definindo extensão do mapa
```{r message=FALSE}
bbox.africa <- st_bbox(africa) 

xrange <- bbox.africa$xmax - bbox.africa$xmin
yrange <- bbox.africa$ymax - bbox.africa$ymin

bbox.africa[1] <- bbox.africa[1] - (0.1 * xrange) 
bbox.africa[3] <- bbox.africa[3] + (0.1 * xrange) 
bbox.africa[2] <- bbox.africa[2] - (0.05 * yrange) 
bbox.africa[4] <- bbox.africa[4] + (0.05 * yrange)

bbox.africa <- bbox.africa %>% st_as_sfc()
```

#### Definindo legendas de texto para os mapas
Legenda para o mapa 1970 - 2000
```{r message=FALSE}
legend_p <- "Dados: WorldClim\nVersão: 2.1\nSistema de Coordenadas Geográficas\nDatum: WGS 84\nResolução espacial: 2,5'"
```

Legenda para o mapa futuro
```{r message=FALSE}
legend_f <- "Dados: WorldClim\nModelo Climático: MIROC6\nSistema de Coordenadas Geográficas\nDatum: WGS 84\nResolução espacial: 2,5'"
```

#### Definindo intervalos da legenda de precipitação
Selecionando como o raster com os maiores valores de precipitação, para definir o intervalo padrão
```{r message=FALSE}
print(mask_presente)
print(mask_futuro)
```

Criando os intervalos da legenda
```{r message=FALSE}
l.max <- max(values(mask_futuro), na.rm = T)
l.min <- min(values(mask_futuro), na.rm = T)
l.mid <- l.max/2
l.int <- l.max/20
```

#### Criando a legenda
```{r message=FALSE, fig.width=10, fig.height=5, warning=FALSE}
legend <- tm_shape(mask_futuro, raster.downsample = F) + # Dados base para a legenda
  tm_raster(palett = "RdYlBu", style = "cont", legend.reverse = T, # Definindo tipo, cores, intervalo e título da legenda
            title = "PAM (mm)", breaks = seq(l.min, l.max, l.int),
            midpoint = l.mid) +
  tm_legend(legend.only = T, scale = 2, asp = 0, # Plotar somente legenda
            legend.title.fontface = "bold",
            legend.stack = "vertical", legend.text.size = 0.5, # Tamanho do texto da legenda
            legend.position = c("center", "center"), # Local de plot da legenda
            legend.format = list(text.align = "center", # Definindo alinhamento do texto na barra de cores
                                 big.mark = ".")) # Definindo separador de milhares
print(legend)
```

#### Criando mapa de preciptação média 1970 - 2000
```{r message=FALSE, fig.width=10, fig.height=5, warning=FALSE}
map_presente <- tm_shape(mask_presente, raster.downsample = F, bbox = bbox.africa) +
  tm_raster(palett = "RdYlBu", style = "cont", legend.reverse = T,
            legend.show = F, breaks = seq(l.min, l.max, l.int), # Desativando legenda
            midpoint = l.mid) +
  tm_graticules(lines = F, n.x = 2, n.y = 3, # Definindo intervalo de exibição das coordenadas no mapa
                labels.rot = c(0, 90), labels.size = 0.75) + # Rotacionando texto da coordenada y e tamanho do texto das coordenadas
  tm_shape(africa) + # Inserindo Contorno dos países
  tm_borders(lwd = 0.1, col = "black") + # Definindo espessura e cor do contorno dos países
  tm_layout(panel.labels = "Presente", # Título interno do mapa
            main.title = "Precipitação Anual Média (PAM) - (1970 - 2000)", # Título externo do mapa
            main.title.position = "center", # Alinhando texto do título
            main.title.size = 1.15, # Tamanho do texto do título
            main.title.fontface = "bold") + # Colocando texto em negrito
  tm_scale_bar(text.size = 1, width = 0.2, position = c("right", "bottom")) + # Inserindo escala no mapa
  tm_compass(type ="8star", size = 4, position = c("right", "top")) + # Inserindo rosa dos ventos no mapa
  tm_text(text = "name_long2", size = 0.5, ) + # Inserindo nome dos países no mapa
  tm_credits(text = legend_p, align = "center", fontface = "bold", # Inserindo legenda de texto com informações para gerar o mapa
             position = c("left", "bottom"), size = 0.64)
print(map_presente)
```

#### Criando mapa de preciptação média 2041 - 2060
```{r message=FALSE, fig.width=10, fig.height=5, warning=FALSE}
map_futuro <- tm_shape(mask_futuro, raster.downsample = F, bbox = bbox.africa) +
  tm_raster(palett = "RdYlBu", style = "cont", legend.reverse = T,
            legend.show = F, breaks = seq(l.min, l.max, l.int),
            midpoint = l.mid) +
  tm_graticules(lines = F, n.x = 2, n.y = 3,
                labels.rot = c(0, 90), labels.size = 0.75) + 
  tm_shape(africa) +
  tm_borders(lwd = 0.1, col = "black") +
  tm_layout(panel.labels = "Futuro, Cenário: SSP126",
            main.title = "Precipitação Anual Média (PAM) - (2041 - 2060)",
            main.title.position = "center",
            main.title.size = 1.15,
            main.title.fontface = "bold") +
  tm_scale_bar(text.size = 1, width = 0.2, position = c("right", "bottom")) +
  tm_compass(type ="8star", size = 4, position = c("right", "top")) +
  tm_text(text = "name_long2", size = 0.5, ) +
  tm_credits(text = legend_f, align = "center", fontface = "bold",
             position = c("left", "bottom"), size = 0.64)
print(map_futuro)
```

#### Criando mapa final, agrupando `1970 - 2000`, `futuro` e `legenda`
```{r message=FALSE, fig.width=10, fig.height=5, warning=FALSE, dpi=300}
map_final <- tmap_arrange(map_presente, map_futuro, legend, ncol = 3,
                           widths = c(0.45, 0.45, 0.1))
print(map_final)
```

<p>&nbsp;</p>

### Salvando mapa final
```{r error=TRUE, message=FALSE, echo=T, eval=F}
tmap_save(map_pronto, dpi = 600, filename = "../mapas/mapa.jpg", width = 12,
          height = 6.75, units = "in")
```

<p>&nbsp;</p>

### Vídeo tutorial
<iframe width="600" height="337.5" src="https://www.youtube.com/embed/lr5bXEs0IbQ" frameborder="0" allowfullscreen></iframe>

<p>&nbsp;</p>
