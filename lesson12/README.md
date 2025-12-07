# Урок "Хранимые функции и процедуры часть 3"

Создаю новую схему и меняю путь по умолчанию:
```pgsql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ;
```

Создаю новые таблицы:
```pgsql
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
```

Создаю таблицу для денормализации:
```pgsql
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```


Создаю функцию и триггер:
```pgsql
CREATE OR REPLACE FUNCTION update_good_sum_mart()
RETURNS TRIGGER AS $$
BEGIN
    -- Обработка операции DELETE
    IF (TG_OP = 'DELETE') THEN
        -- Уменьшаем сумму в таблице good_sum_mart
        UPDATE good_sum_mart gsm
        SET sum_sale = gsm.sum_sale - (
            SELECT g.good_price * OLD.sales_qty
            FROM goods g
            WHERE g.goods_id = OLD.good_id
        )
        WHERE gsm.good_name = (
            SELECT g.good_name
            FROM goods g
            WHERE g.goods_id = OLD.good_id
        );
        
        -- Если сумма стала 0 или меньше, удаляем запись
        DELETE FROM good_sum_mart 
        WHERE sum_sale <= 0;
        
        RETURN OLD;
    
    -- Обработка операции UPDATE
    ELSIF (TG_OP = 'UPDATE') THEN
        -- Если изменился товар или количество
        IF OLD.good_id != NEW.good_id OR OLD.sales_qty != NEW.sales_qty THEN
            -- Для старого товара уменьшаем сумму
            UPDATE good_sum_mart gsm
            SET sum_sale = gsm.sum_sale - (
                SELECT g.good_price * OLD.sales_qty
                FROM goods g
                WHERE g.goods_id = OLD.good_id
            )
            WHERE gsm.good_name = (
                SELECT g.good_name
                FROM goods g
                WHERE g.goods_id = OLD.good_id
            );
            
            -- Для нового товара добавляем сумму
            -- Проверяем, существует ли уже запись для этого товара
            IF EXISTS (
                SELECT 1 FROM good_sum_mart 
                WHERE good_name = (
                    SELECT g.good_name FROM goods g WHERE g.goods_id = NEW.good_id
                )
            ) THEN
                -- Если существует, обновляем сумму
                UPDATE good_sum_mart gsm
                SET sum_sale = gsm.sum_sale + (
                    SELECT g.good_price * NEW.sales_qty
                    FROM goods g
                    WHERE g.goods_id = NEW.good_id
                )
                WHERE gsm.good_name = (
                    SELECT g.good_name
                    FROM goods g
                    WHERE g.goods_id = NEW.good_id
                );
            ELSE
                -- Если не существует, вставляем новую запись
                INSERT INTO good_sum_mart (good_name, sum_sale)
                SELECT 
                    g.good_name,
                    g.good_price * NEW.sales_qty
                FROM goods g
                WHERE g.goods_id = NEW.good_id;
            END IF;
            
            -- Удаляем запись если сумма стала 0 или меньше для старого товара
            DELETE FROM good_sum_mart 
            WHERE sum_sale <= 0;
        END IF;
        RETURN NEW;
    
    -- Обработка операции INSERT
    ELSIF (TG_OP = 'INSERT') THEN
        -- Проверяем, существует ли уже запись для этого товара
        IF EXISTS (
            SELECT 1 FROM good_sum_mart 
            WHERE good_name = (
                SELECT g.good_name FROM goods g WHERE g.goods_id = NEW.good_id
            )
        ) THEN
            -- Если существует, обновляем сумму
            UPDATE good_sum_mart gsm
            SET sum_sale = gsm.sum_sale + (
                SELECT g.good_price * NEW.sales_qty
                FROM goods g
                WHERE g.goods_id = NEW.good_id
            )
            WHERE gsm.good_name = (
                SELECT g.good_name
                FROM goods g
                WHERE g.goods_id = NEW.good_id
            );
        ELSE
            -- Если не существует, вставляем новую запись
            INSERT INTO good_sum_mart (good_name, sum_sale)
            SELECT 
                g.good_name,
                g.good_price * NEW.sales_qty
            FROM goods g
            WHERE g.goods_id = NEW.good_id;
        END IF;
        
        RETURN NEW;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;


CREATE OR REPLACE TRIGGER sales_good_sum_mart_trigger
AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW
EXECUTE FUNCTION update_good_sum_mart();
```

Вставляю данные в sales и проверяю заполнение таблицы good_sum_mart:
```pgsql
INSERT INTO sales (good_id, sales_qty) VALUES (1, 500);
INSERT INTO sales (good_id, sales_qty) VALUES (2, 500);
```

В ответ получаю:
```text
Спички хозайственные	250.00
Автомобиль Ferrari FXX K	92500000005.00
```
