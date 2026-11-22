# Karta Tesla dla Home Assistant (TeslaMate + MQTT + button-card)

Dashboard z jedną kartą `custom:button-card`, który pokazuje stan auta Tesla na podstawie danych z [TeslaMate](https://github.com/teslamate-org/teslamate) publikowanych przez MQTT. Nie korzysta z Tesla Fleet API ani z żadnego zewnętrznego API. Czyta wyłącznie encje HA zdefiniowane w `mqtt_tesla.yaml`.

Pokazuje:

- stan auta (online, uśpione, jazda, ładowanie), bieg, prędkość i lokalizację (geofence),
- poziom baterii z markerem limitu ładowania, zasięg, przebieg, temperatury,
- panel ładowania (moc, prąd, napięcie, dodana energia, czas do pełna),
- widok auta z góry: ciśnienie w każdej oponie, stan czterech drzwi, frunk i bagażnik,
- zamek, Sentry, klimatyzację, okna (stan zbiorczy), port ładowania, obecność w aucie, wersję oprogramowania.

## Wymagania

- Home Assistant z integracją MQTT i brokerem (np. dodatek Mosquitto).
- TeslaMate publikujący na tym brokerze (zmienne `MQTT_HOST`, `MQTT_USERNAME`, `MQTT_PASSWORD`).
- [button-card](https://github.com/custom-cards/button-card) zainstalowany przez HACS (testowane z 7.0.1).
- Dwa zdjęcia auta. Przykładowe są w `www/tesla/`, możesz użyć własnych.

## Ograniczenia

- Okna są pokazane jednym polem „Okna”. TeslaMate publikuje tylko zbiorczy temat `windows_open`, bez stanu poszczególnych szyb.
- Dane są takie, jak publikuje TeslaMate. Gdy auto śpi, wartości mogą być nieaktualne.
- Plik zakłada `car_id` równe `1` (tematy `teslamate/cars/1/...`). Przy innym numerze zamień `cars/1` w całym pliku `mqtt_tesla.yaml`.
- Karta jest dopasowana do sylwetki Modelu 3 na zdjęciu z góry o proporcjach 443×700 px. Przy innym zdjęciu trzeba będzie przesunąć ramki opon i paski drzwi (reguły `.tc-tire`, `.tc-door` w `dashboards/tesla.yaml`).
- Układ sprawdziłem tylko automatycznie (render bez błędów wykonania) i na jednej instalacji. Nie testowałem go na wielu urządzeniach i motywach.

## Instalacja

1. **Zainstaluj button-card** z HACS (Frontend) i odśwież przeglądarkę.

2. **Skopiuj pliki** do katalogu konfiguracji HA (`/config`):

   ```
   mqtt_tesla.yaml          -> /config/mqtt_tesla.yaml
   dashboards/tesla.yaml    -> /config/dashboards/tesla.yaml
   ```

3. **Skopiuj zdjęcia auta** z `www/tesla/` do `/config/www/tesla/` (adres w HA: `/local/tesla/...`):

   | Plik | Co przedstawia | Rozmiar |
   |---|---|---|
   | `tesla-iso.jpg` | auto z przodu pod kątem, ciemne tło `#161719` | 800×420 px |
   | `tesla-top.jpg` | auto z góry, pionowo, przodem do góry, to samo tło | 443×700 px |

   Przykładowe zdjęcia przedstawiają czerwony Model 3. Są zrzutami ekranu, z których usunięto elementy interfejsu. Przed ponownym wykorzystaniem sprawdź, czy możesz ich używać, np. we własnych celach prywatnych. Własne zdjęcia muszą mieć te same proporcje i nazwy. Układ ramek opon i drzwi jest dopasowany do sylwetki z `tesla-top.jpg`.

4. **Dołącz encje MQTT** w `configuration.yaml`.

   Jeśli nie masz jeszcze sekcji `mqtt:`:

   ```yaml
   mqtt: !include mqtt_tesla.yaml
   ```

   Jeśli już ją masz, dopisz zawartość `mqtt_tesla.yaml` do własnych list `sensor:` i `binary_sensor:`.

5. **Zarejestruj dashboard** w `configuration.yaml`:

   ```yaml
   lovelace:
     mode: storage
     dashboards:
       tesla-yaml:
         mode: yaml
         title: Tesla
         icon: mdi:car-electric
         show_in_sidebar: true
         filename: dashboards/tesla.yaml
   ```

   Jeśli masz już sekcję `lovelace:`, dodaj tylko wpis w `dashboards:`.

6. **Sprawdź konfigurację i zrestartuj HA.** W *Narzędzia deweloperskie → YAML → Sprawdź konfigurację* lub komendą `ha core check`. Zmiana `configuration.yaml` wymaga restartu. Późniejsze zmiany w `dashboards/tesla.yaml` wymagają tylko odświeżenia strony (Cmd/Ctrl+Shift+R).

7. **Otwórz** pozycję „Tesla” w pasku bocznym.

## Nazwy encji

Wszystkie encje mają stałe `default_entity_id`, więc dashboard znajdzie je niezależnie od języka HA. Przykłady: `sensor.tesla_tesla_poziom_baterii`, `sensor.tesla_tpms_fl`, `binary_sensor.tesla_door_fl`, `binary_sensor.tesla_windows_open`. Pełna lista jest w `mqtt_tesla.yaml`.

Jeśli masz już encje o takich `unique_id` lub nazwach, HA może dodać sufiks `_2`. Usuń wtedy stare encje w *Ustawienia → Urządzenia i usługi → Encje* i uruchom ponownie HA.

## Bezpieczeństwo

- Hasła do MQTT trzymaj w pliku `.env` TeslaMate (uprawnienia `600`) i w menedżerze użytkowników brokera. Nie wkładaj ich do tego repozytorium ani do `configuration.yaml`.
- Konto MQTT dla TeslaMate warto ograniczyć w ACL brokera do zapisu w `teslamate/#`.
- Porty TeslaMate (4000) i Grafany (3000) nie mają domyślnie uwierzytelniania. Nie wystawiaj ich do internetu i nie wiąż ich z `0.0.0.0` bez reverse proxy z logowaniem.
- MQTT w LAN bez TLS przesyła dane jawnym tekstem. Jeśli broker jest dostępny poza zaufaną siecią, włącz TLS.

## Rozwiązywanie problemów

| Objaw | Przyczyna i działanie |
|---|---|
| Karta pokazuje błąd szablonu lub jest pusta | Sprawdź konsolę przeglądarki. Szablon JS w button-card nie może deklarować zmiennych o nazwach `states`, `entity`, `user`, `hass`, `variables`, `html`, `helpers`. |
| Wszędzie „–” | Encje są `unknown`. Sprawdź w MQTT Explorerze, czy `teslamate/cars/1/#` ma wartości. |
| Brak zdjęć | Pliki muszą leżeć w `/config/www/tesla/` i mieć dokładnie takie nazwy jak wyżej. |
| Zmiana w `tesla.yaml` bez efektu | Odśwież z pominięciem pamięci podręcznej (Cmd/Ctrl+Shift+R). |

## Licencja

MIT, zobacz [LICENSE](LICENSE).
