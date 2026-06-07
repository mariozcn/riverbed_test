# NOTES — Mario Rusu

Salut! Mai jos am scris cum am abordat task-ul.

## Cum am inceput

Primul lucru pe care l-am facut a fost sa rulez `pytest -v`, ca sa vad unde stau.
Imi picau 6 teste. Mi s-a parut mai logic sa pornesc de la testele rosii inapoi
spre cod, decat sa citesc tot codul de la zero (oricum e destul de putin, l-am citit
si pe ala dupa).

## 1. Bug-urile pe care le-am gasit

### Bug 1 — paginarea sarea peste primul eveniment
- **Unde:** `app/storage.py`, in `list_events`.
- **Cum l-am prins:** `test_list_events_includes_created_items` voia 5 evenimente si
  primea 4. M-am uitat in `list_events` si slice-ul era `[offset + 1 : offset + 1 + limit]`.
  `+ 1` ala sarea mereu peste primul element. La inceput nu mi-am dat seama de ce,
  am pus un `print` si am vazut ca lipseste mereu primul.
- **Fix:** am scos `+ 1`, acum e `[offset : offset + limit]`.

### Bug 2 — evenimentele sterse tot apareau in lista
- **Unde:** tot in `list_events`.
- **Cum l-am prins:** `test_list_events_hides_soft_deleted_items` pica. Am vazut ca
  `delete` doar seteaza `deleted_at`, dar `list_events` returna tot, inclusiv ce era
  sters. Practic soft-delete-ul nu era luat in calcul nicaieri la citire.
- **Fix:** filtrez inainte de paginare cu `e.deleted_at is None`.

### Bug 3 — puteai sterge acelasi eveniment de doua ori
- **Unde:** `app/storage.py`, `soft_delete_event`.
- **Cum l-am prins:** `test_delete_same_event_twice_changes_response` astepta 204
  prima data si 404 a doua oara, dar primea 204 de doua ori. Metoda rescria `deleted_at`
  fara sa verifice daca era deja sters.
- **Fix:** daca evenimentul e deja sters (`deleted_at is not None`) returnez `None`,
  iar endpoint-ul transforma asta in 404.

### Bug 4 — `POST /events` returna 200 in loc de 201
- **Unde:** `app/main.py`, decoratorul lui `create_event`.
- **Cum l-am prins:** `test_create_event_returns_201` primea 200. Am observat ca la
  `create_user` era deja `status_code=201`, dar la evenimente nu — deci era doar uitat.
- **Fix:** am adaugat `status_code=201` la decorator.

## 2. Endpoint-ul nou — `GET /users/{user_id}/events?since=<ISO_date>`

Am incercat sa-l fac sa arate ca restul codului: endpoint subtire in `main.py`, iar
logica in `storage.list_user_events`.

Decizii pe care le-am luat:
- Am pus `since` ca `Optional[datetime]`, ca sa lase FastAPI sa parseze data ISO
  singur. Bonus: daca trimiti o data invalida, da 422 automat, fara sa scriu eu cod.
- "Mai noi decat data" l-am luat strict, adica `created_at > since`.
- Verific userul intai — daca nu exista, 404 (nu vreau sa returnez lista goala pentru
  un user care nici nu exista).
- Am ascuns si aici evenimentele sterse, ca sa se comporte la fel ca `GET /events`.

Partea la care m-am impiedicat: daca trimiti `since` fara timezone (ex. doar
`2026-01-01`), Python crapa cand compari un datetime "naive" cu unul "aware". Nu
stiam de chestia asta, mi-a luat ceva sa-mi dau seama de ce pica. Solutia pe care
am ales-o e sa tratez un `since` fara timezone ca UTC.

Teste pe care le-am adaugat (4):
- `test_user_events_returns_only_that_users_events` — sa nu amestece userii.
- `test_user_events_missing_user_returns_404` — user inexistent da 404.
- `test_user_events_filters_by_since` — `since` taie evenimentele mai vechi.
- `test_user_events_excludes_soft_deleted` — ce e sters nu apare.

La final toate cele 16 teste trec.

## 3. Unde m-a ajutat AI-ul

Am folosit Claude, cam cum fac si la facultate cand ma blochez.
- M-a ajutat cel mai mult la capcana cu datetime naive vs. aware — nu auzisem de ea
  si nu intelegeam eroarea, mi-a explicat ce se intampla.
- L-am folosit si ca sa-mi confirm ca interpretarea cu `>` (strict) e rezonabila.
- Unde a trebuit sa fiu atent: prima varianta de filtru pentru `since` pe care mi-a
  dat-o nu trata cazul fara timezone. Am observat cand m-am gandit efectiv cum se
  compara datele, si am adaugat normalizarea la UTC.

Tot codul pe care l-am livrat pot sa-l explic, n-am pus nimic ce nu inteleg.
