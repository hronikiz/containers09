# Лабораторная работа №9  
## Оптимизация Docker-образов

---

## 1. Название лабораторной работы
Оптимизация Docker-образов различными методами

---

## 2. Цель работы
Целью работы является изучение и сравнение методов оптимизации Docker-образов, а также анализ их влияния на размер и структуру образов.

---

## 3. Задание
В рамках лабораторной работы необходимо:

- Изучить Docker-образы и их структуру
- Сравнить методы оптимизации:
  - удаление неиспользуемых зависимостей и временных файлов
  - уменьшение количества слоев
  - использование минимального базового образа
  - перепаковка образа
  - комбинирование всех методов
- Проанализировать влияние методов на размер образа
- Сделать выводы

---

## 4. Выполнение работы

### 4.1 Подготовка проекта

Был создан репозиторий `containers09` и структура проекта:

```
containers09/
 ├── site/
 │    ├── index.html
 │    ├── style.css
 │    └── script.js
 ├── Dockerfile.raw
 ├── Dockerfile.clean
 ├── Dockerfile.few
 ├── Dockerfile.alpine
 ├── Dockerfile.min
 └── README.md
```

<img width="310" height="295" alt="image" src="https://github.com/user-attachments/assets/d83496de-c8ee-4814-b13a-10aa68c1a9e3" />

Сайт содержит простой HTML/CSS/JS интерфейс.

---

## 4.2 Базовый образ (RAW)

Использовался Dockerfile на базе Ubuntu:

```dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y nginx

COPY site /var/www/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Результат сборки:
- `mynginx:raw`
<img width="743" height="18" alt="image" src="https://github.com/user-attachments/assets/4f9a791f-b276-462e-b1b1-3b94967fc030" />

---

## 4.3 Удаление кеша (CLEAN)

Добавлена очистка временных файлов:

```dockerfile
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Результат:
- `mynginx:clean`
<img width="687" height="20" alt="image" src="https://github.com/user-attachments/assets/763e809c-0090-41c7-a8eb-d1e1385c7b35" />

---

## 4.4 Уменьшение количества слоев

Объединены команды RUN:

```dockerfile
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y nginx && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Результат:
- `mynginx:few`
<img width="673" height="20" alt="image" src="https://github.com/user-attachments/assets/6328049a-786b-4165-8749-052e401b823e" />

---

## 4.5 Минимальный базовый образ (Alpine)

Использован легковесный образ:

```dockerfile
FROM alpine:latest

RUN apk update && apk upgrade
RUN apk add nginx
```

Результат:
- `mynginx:alpine`
<img width="682" height="23" alt="image" src="https://github.com/user-attachments/assets/86bb9b70-75b5-4fcd-a477-bacad320d8a7" />

---

## 4.6 Полная оптимизация

Комбинация всех методов:

```dockerfile
FROM alpine:latest

RUN apk update && apk upgrade && \
    apk add nginx && \
    rm -rf /var/cache/apk/*

COPY site /var/www/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Результат:
- `mynginx:minx`
<img width="724" height="28" alt="image" src="https://github.com/user-attachments/assets/b5e76391-3ac4-4597-a14d-65d1644960eb" />

---

## 4.7 Перепаковка образа

Перепаковка выполнялась через экспорт контейнера:

```bash
docker container create --name mynginx mynginx:minx
docker container export mynginx -o mynginx.tar
docker image import mynginx.tar mynginx:min
docker container rm mynginx
```

Результат:
- `mynginx:min`
<img width="692" height="21" alt="image" src="https://github.com/user-attachments/assets/16b1c78b-8f26-4e0f-8e6e-6041884fa6ca" />

---

## 5. Результаты измерений

| Образ            | Метод оптимизации                     | Disk Usage | Content Size |
|------------------|--------------------------------------|------------|--------------|
| mynginx:raw      | Ubuntu + nginx                       | 267MB      | 84.7MB       |
| mynginx:clean    | Очистка кеша                        | 267MB      | 84.7MB       |
| mynginx:few      | Уменьшение слоев                    | 162MB      | 41.7MB       |
| mynginx:alpine   | Минимальный базовый образ          | 22MB       | 7.66MB       |
| mynginx:minx     | Полная оптимизация                  | 16.1MB     | 4.74MB       |
| mynginx:min      | Перепакованный образ               | 16MB       | 4.73MB       |
| mynginx:repack   | Перепаковка RAW                    | 232MB      | 75.9MB       |

---
<img width="802" height="428" alt="image" src="https://github.com/user-attachments/assets/e7a41f84-9ff7-4bd8-b013-4d1a2d5e8de3" />

## 6. Анализ результатов

- Самый большой размер имеет образ на базе Ubuntu (`raw`)
- Очистка кеша не уменьшила размер, так как Docker хранит данные в слоях
- Уменьшение количества слоев дало умеренное улучшение
- Использование Alpine дало наибольшее снижение размера
- Перепаковка удаляет слои, но не всегда значительно уменьшает размер

---

## 7. Ответы на вопросы

### 1. Какой метод наиболее эффективный?
Наиболее эффективным методом является использование минимального базового образа (Alpine), так как он изначально содержит минимальный набор системных компонентов.

---

### 2. Почему очистка кеша не уменьшает размер?
Потому что Docker использует слоистую систему хранения. Данные, добавленные в одном слое, сохраняются даже если они удалены в следующем слое.

---

### 3. Что такое перепаковка образа?
Перепаковка образа — это процесс создания контейнера, экспорта его файловой системы и последующего импорта как нового образа без истории слоев.

---

## 8. Вывод

В ходе лабораторной работы были изучены методы оптимизации Docker-образов.  
Наиболее значительное уменьшение размера достигается за счет использования минимального базового образа.  
Комбинирование методов позволяет получить максимально компактный образ.  

---
