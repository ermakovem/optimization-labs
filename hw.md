I. MaxMemory.java

1. -Xmx512m

	`Max memory: 460 MB`

-Xmx задаёт максимальный размер кучи, поэтому размер стал равен 512 МБ.

2. -Xmx512m -Xms512m

	`Max memory: 493 MB`

-Xms задаёт начальный (минимальный) размер кучи. Разница лишь в том, что теперь начальная куча (-Xms) тоже 512 МБ, 
т.е. при старте JVM сразу выделит полный объём, но метод maxMemory() всё так же покажет ту же верхнюю границу.

3. -Xmx512m -Xms512m -XX:+PrintGCDetails

`[0.002s][warning][gc] -XX:+PrintGCDetails is deprecated. Will use -Xlog:gc* instead.
[0.005s][info   ][gc,init] CardTable entry size: 512
[0.005s][info   ][gc     ] Using G1
[0.007s][info   ][gc,init] Version: 23.0.1+11-39 (release)
[0.007s][info   ][gc,init] CPUs: 28 total, 28 available
[0.007s][info   ][gc,init] Memory: 32581M
[0.007s][info   ][gc,init] Large Page Support: Disabled
[0.007s][info   ][gc,init] NUMA Support: Disabled
[0.007s][info   ][gc,init] Compressed Oops: Enabled (32-bit)
[0.007s][info   ][gc,init] Heap Region Size: 1M
[0.007s][info   ][gc,init] Heap Min Capacity: 512M
[0.007s][info   ][gc,init] Heap Initial Capacity: 512M
[0.007s][info   ][gc,init] Heap Max Capacity: 512M
[0.007s][info   ][gc,init] Pre-touch: Disabled
[0.007s][info   ][gc,init] Parallel Workers: 20
[0.007s][info   ][gc,init] Concurrent Workers: 5
[0.007s][info   ][gc,init] Concurrent Refinement Workers: 20
[0.007s][info   ][gc,init] Periodic GC: Disabled
[0.013s][info   ][gc,metaspace] CDS archive(s) mapped at: [0x0000019a80000000-0x0000019a80d70000-0x0000019a80d70000), size 14090240, SharedBaseAddress: 0x0000019a80000000, ArchiveRelocationMode: 1.
[0.013s][info   ][gc,metaspace] Compressed class space mapped at: 0x0000019a81000000-0x0000019ac1000000, reserved size: 1073741824
[0.013s][info   ][gc,metaspace] Narrow klass base: 0x0000019a80000000, Narrow klass shift: 0, Narrow klass range: 0x41000000
Max memory: 512 MB[0.090s][info   ][gc,heap,exit] Heap
[0.090s][info   ][gc,heap,exit]  garbage-first heap   total reserved 524288K, committed 524288K, used 8192K [0x00000000e0000000, 0x0000000100000000)
[0.090s][info   ][gc,heap,exit]   region size 1024K, 8 young (8192K), 0 survivors (0K)
[0.090s][info   ][gc,heap,exit]  Metaspace       used 993K, committed 1216K, reserved 1114112K
[0.090s][info   ][gc,heap,exit]   class space    used 77K, committed 192K, reserved 1048576K`

Using G1 — включён сборщик мусора G1;
Heap Region Size: 1M — G1 дробит кучу на регионы по 1 МБ каждый;
Heap Min Capacity / Initial Capacity / Max Capacity = 512M — из-за -Xms512m -Xmx512m начальный и максимальный размер кучи совпадают (512 МБ).

Видим, что G1 GC инициализировался с кучей в 512 МБ (как мы и задали).
JVM успела создать несколько десятков классов (JDK-классы + класс MaxMemory), занять чуть меньше 1 МБ в Metaspace + 8 МБ в куче.
«Max memory: 512 MB» подтверждает, что Runtime.getRuntime().maxMemory() даёт ровно 512.
Строки после [gc,heap,exit] и [gc,metaspace] — это сводка: сколько памяти было фактически занято и как она была поделена на регионы.

Также видно, что на машине 28 доступных процессоров, системная оперативная память ~32 ГБ (Memory: 32581M), но самой JVM под кучу выделено максимум 512 МБ согласно флагам.

4. -Xms512m -Xmx512m -XX:SurvivorRatio=100
  	`Max memory: 510 MB`
5. -Xmx512m -XX:+UseG1GC
    `Max memory: 512 MB`
6. 
II. Phantom references
1. Без -Dphantom.refs=true, при маленькой куче 24 МБ.
Почти нет частых сборок мусора, потому что в итоге не расходуется память на новые объекты (по сути, все PhantomReference указывают на один и тот же substitute).

2. С -Dphantom.refs=true, при куче 24 МБ.
Создаётся множество короткоживущих объектов, каждый завернут в PhantomReference. Память быстро заполняется -> частые GC.

3. С -Dphantom.refs=true, но уже при куче 64 МБ
Та же логика, что и в п. 2, но из-за большего размера кучи сборки мусора происходят реже.

4. С -Dphantom.refs=true и -Dno.ref.clearing=true, при куче 64 МБ
Объекты всё так же живут недолго, но сами PhantomReference не удаляются из ReferenceQueue, копятся и занимают память. Итог – либо чаще GC, либо при длительном запуске возможна утечка (OutOfMemoryError), если очередь растёт быстрее, чем сборщик успевает что-либо очистить.

III. Premature promotion
1. Default heap (не указано -Xmx), MaxTenuringThreshold=1.
Сборок Minor будет какое-то среднее количество.
Full GC зависит от того, насколько на самом деле большая (или маленькая) дефолтная куча. Часто первых 15–30 с может хватить, чтобы не достичь Full GC, но это зависит от машины.

2. C -Xmx24m, New=16m, MaxTenuringThreshold=1
Из-за маленькой кучи будут частые Minor GC, а из-за маленького OldGen (≈8 МБ) и низкого тенуринга — частые Full GC при быстром наполнении OldGen.

3. -Xmx64m, New=32m, MaxTenuringThreshold=1
Больше памяти -> реже Full GC (или вообще может не случиться за 15–30 с).
Minor GC будут происходить регулярно, но реже, чем в п. 2 (т.к. Eden 32 МБ).

4. -Dmax.chunks=1000, -Xmx24m, New=16m, MaxTenuringThreshold=1
Снова маленькая куча, но быстреее очищаются ссылки (пороги 1000 вместо 10000).
Много Minor GC, но Full GC может быть меньше, чем в пункте 2, потому что объекты не успевают долго оставаться в accumulatedChunks.

5. -Xmx64m, New=32m, +NeverTenure
Не продвигаем объекты в OldGen.
Очень частые Minor GC, чтобы чистить Eden/Survivor.
При переполнении молодых поколений живыми объектами может срабатывать Full GC (хотя по идее NeverTenure пытается всё держать в YoungGen, на практике при очень сильном заполнении возможны Full GC).


IV. Soft references

1. Без -Dsoft.refs=true.
Мы фактически используем один и тот же объект substitute для всех SoftReference, поэтому памяти уходит совсем мало. Сборок мусора (особенно Full GC) будет минимум.

2. С -Dsoft.refs=true, но при -Xmx24m
Каждый новый объект действительно живёт в SoftReference.
Память небольшая, Eden быстро заполняется: результат — частые Minor GC, а при попытках переместить живые объекты в OldGen (около 8 МБ) иногда может доходить и до Full GC.
Однако SoftReference будут убиваться сборщиком мусора при нехватке памяти, так что часто объекты даже не дойдут до OldGen, а будут сброшены в Minor GC.

3. С -Dsoft.refs=true при -Xmx64m
Больше памяти (64 МБ, из них 32 МБ в YoungGen).
Minor GC происходят реже, Full GC может вообще не наблюдаться в первые 15–30 с, так как GC имеет больше пространства, а при нехватке — мягкие ссылки очистятся раньше, чем случится полный сбор.

V. Weak references

1. Без -Dweak.refs=true
Все WeakReference указывают на один объект substitute.
Памть расходуется минимально, сборок мусора (особенно Full GC) практически нет.

2. С -Dweak.refs=true, -Xmx24m:
Каждый объект живет в WeakReference и легко освобождается на Minor GC.
Из-за маленького YoungGen (16 МБ) объекты быстро заполняют его -> частые Minor GC, но объекты удляются сразу, так что OldGen почти не растёт -> Full GC либо мало, либо нет

3. С -Dweak.refs=true, -Xmx64m:
Та же ситуация со слабыми ссылками, но YoungGen больше (32 МБ) -> Minor GC реже, Full GC практически не встречается, объекты быстро собираются при первой Minor GC после потери сильной ссылки.

