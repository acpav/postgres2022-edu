# Домашнее задание 08  
## Настройка autovacuum с учетом оптимальной производительности  

На свежий инстанс установил PostgreSQL 13 с дефолтными настройками  
Применил настройки из приложенного файла  

Выполнил **pgbench -i postgres** и запустил **pgbench -c8 -P 60 -T 3600 -U postgres postgres**  

Значения за последние 10 минут  

<code>
progress: 3000.0 s, 503.2 tps, lat 15.862 ms stddev 13.391  
progress: 3060.0 s, 444.9 tps, lat 17.956 ms stddev 16.908  
progress: 3120.0 s, 560.9 tps, lat 14.232 ms stddev 12.255  
progress: 3180.0 s, 540.3 tps, lat 14.771 ms stddev 12.888  
progress: 3240.0 s, 521.4 tps, lat 15.316 ms stddev 13.383  
progress: 3300.0 s, 580.3 tps, lat 13.756 ms stddev 11.733  
progress: 3360.0 s, 584.7 tps, lat 13.651 ms stddev 11.900  
progress: 3420.0 s, 548.8 tps, lat 14.546 ms stddev 12.919  
progress: 3480.0 s, 592.4 tps, lat 13.472 ms stddev 11.752  
progress: 3540.0 s, 562.1 tps, lat 14.197 ms stddev 12.161  
progress: 3600.0 s, 503.9 tps, lat 15.846 ms stddev 14.096  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 3600 s  
number of transactions actually processed: 1877497  
latency average = 15.308 ms  
latency stddev = 14.867 ms  
tps = 521.518793 (including connections establishing)  
tps = 521.519093 (excluding connections establishing)  
</code>

Был на следующий день, после перезапуска сервера  
Применил настройки  

<code>
autovacuum = on  
log_autovacuum_min_duration = -1  
autovacuum_max_workers = 2  --На инстансе всего 2 ядра, больше 2-х воркеров нет смысла делать
autovacuum_naptime = 30s  
autovacuum_vacuum_threshold = 300  
autovacuum_vacuum_insert_threshold = -1  
autovacuum_analyze_threshold = 300  
autovacuum_vacuum_scale_factor = 0.05  
autovacuum_analyze_scale_factor = 0.05  
autovacuum_vacuum_cost_delay = 20  
autovacuum_vacuum_cost_limit = 5000  
</dev>

Значения за посление 10 минут  
<dev>
progress: 3000.0 s, 487.8 tps, lat 16.367 ms stddev 14.325  
progress: 3060.0 s, 565.3 tps, lat 14.126 ms stddev 12.503  
progress: 3120.0 s, 607.6 tps, lat 13.136 ms stddev 11.224  
progress: 3180.0 s, 594.3 tps, lat 13.431 ms stddev 11.845  
progress: 3240.0 s, 577.7 tps, lat 13.817 ms stddev 12.322  
progress: 3300.0 s, 394.3 tps, lat 20.264 ms stddev 18.031  
progress: 3360.0 s, 547.6 tps, lat 14.578 ms stddev 12.838  
progress: 3420.0 s, 558.5 tps, lat 14.296 ms stddev 12.841  
progress: 3480.0 s, 506.4 tps, lat 15.769 ms stddev 14.164  
progress: 3540.0 s, 524.1 tps, lat 15.233 ms stddev 13.099  
progress: 3600.0 s, 537.1 tps, lat 14.864 ms stddev 12.816  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 3600 s  
number of transactions actually processed: 1907028  
latency average = 15.073 ms  
latency stddev = 13.747 ms  
tps = 529.724640 (including connections establishing)  
tps = 529.724994 (excluding connections establishing) 
</code>