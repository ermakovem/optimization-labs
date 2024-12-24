I. 

II. Phantom references
1. Без -Dphantom.refs=true, при маленькой куче 24 МБ.
Почти нет частых сборок мусора, потому что в итоге «не расходуется» память на новые объекты (по сути, все PhantomReference указывают на один и тот же substitute).

2. С -Dphantom.refs=true, при куче 24 МБ.
Создаётся множество короткоживущих объектов, каждый завернут в PhantomReference. Память быстро заполняется → частые GC.

3. С -Dphantom.refs=true, но уже при куче 64 МБ.
Та же логика, что и в п. 2, но из-за большего размера кучи сборки мусора происходят реже.

4. С -Dphantom.refs=true и -Dno.ref.clearing=true, при куче 64 МБ.
Объекты всё так же живут недолго, но сами PhantomReference не удаляются из ReferenceQueue, копятся и занимают память. Итог – либо чаще GC, либо при длительном запуске возможна утечка (OutOfMemoryError), если очередь растёт быстрее, чем сборщик успевает что-либо очистить.

III. Premature promotion
1. Без -Xmx, MaxTenuringThreshold=1. Куча большая (по умолчанию), будут нормальные Minor GC, иногда Full GC.
(-Xmx24m, NewSize=16m, MaxTenuringThreshold=1): маленькая куча, частые Minor GC, при заполнении Old Gen (8 МБ) — Full GC.
(-Xmx64m, NewSize=32m, MaxTenuringThreshold=1, лог в файл): больше памяти, Minor GC реже, Full GC тоже реже, но в деталях всё видно в gc.txt.
(-Dmax.chunks=1000, -Xmx24m, NewSize=16m): за счёт быстрого сброса массивов после накопления 1000 шт объекты живут короче, Old Gen заполняется меньше → меньше Full GC.
(-Xmx64m, NewSize=32m, -XX:+NeverTenure): объекты практически не промотируются, всё живёт в Young Gen → очень частые Minor GC, почти нет Full GC.


IV. Soft references



V. Weak references

