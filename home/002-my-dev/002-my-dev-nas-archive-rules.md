# Здесь изложена структура архива личных данных

### L1

```
/Archive
│
├── 00_Inbox
├── 01_Personal
├── 02_Family
├── 03_Partner
├── 04_Shared
├── 06_Media
├── 09_Archive_Cold
```

- **00_Inbox** — входящая зона (всё новое)
- **01_Personal** — только твои данные
- **02_Family** — общее семейное
- **03_Partner** — данные жены
- **04_Shared** — пересечение (общие документы)
- **06_Media** — фото/видео (централизовано)
- **09_Archive_Cold** — неактуальный архив

можно кодировать смысл:
- `00` → вход
- `01–04` → активные данные
- `06` → тяжёлые данные (media)
- `09` → архив

---
### L2  -Denis

```
01_Personal
│
├── Identity
├── Finance
├── Health
├── Work
├── Education
├── Books
├── Knowledge
├── Projects_Artifacts
```

- Identity
	- паспорт
	- визы
	- страховки
	- сертификаты личности
-  Finance - Финансовая история
	- договоры
	- налоги
	- счета
	- зарплата
- Health - Медицина
	- анализы
	- заключения
	- назначения
- Work - Вся профессиональная история
	- резюме
	- офферы
	- рабочие документы
	- переписка/экспорт при необходимости
- Education - Формальное обучение (“структурированное обучение”)
	- университет
	- курсы (с сертификатами)
	- учебные материалы
- Books
	- PDF / EPUB
	- возможно разделение потом по темам
- Knowledge - Неструктурированные знания
	- статьи
	- сохранённые материалы
	- PDF, на которые ссылается Obsidian
	- выгрузки из интернета
- Projects_Artifacts
	- bitstream
	- отчёты
	- скриншоты
	- результаты тестов
	- финальные PDF/доки

---
### L3 -Denis

# Практический стандарт по типам файлов
---
## Identity

Формат:
```
YYYY-MM-DD_document_subject.ext
```
Примеры:
```
2024-05-18_passport-scan.pdf  
2025-09-03_residence-permit_front.jpg  
2025-09-03_residence-permit_back.jpg  
2026-01-12_health-insurance-policy.pdf
```
Если документ без даты выдачи, можно без даты в начале:
```
passport-scan.pdf
```
Но для архива лучше ставить дату сканирования или дату документа.

---
## Finance

Формат:
```
YYYY-MM-DD_document_counterparty_optional.ext
```
Примеры:
```
2026-02-01_salary_company-name.pdf  
2026-03-15_invoice_pcbway.pdf  
2025-12-31_tax-return_germany.pdf  
2026-01-10_rent-payment_apartment-berlin.pdf
```
Если это серия однотипных файлов, можно использовать период:
```
2026-03_bank-statement_commerzbank.pdf  
2026-Q1_tax-documents.pdf
```
---
## Health

Формат:
```
YYYY-MM-DD_document_clinic-or-topic.ext
```
Примеры:
```
2026-03-18_blood-test_labcorp.pdf  
2026-03-21_ultrasound_abdomen.pdf  
2026-03-25_prescription_ursosan.pdf
```
---

## Work

Так как нужно хранить **всю профессиональную историю**, здесь нужен чуть более строгий стиль.

Формат:
```
YYYY-MM-DD_document_company_or_role.ext
```
Примеры:
```
2026-04-01_resume_fpga-prototyping-engineer.pdf  
2026-03-20_cover-letter_amd.pdf  
2025-11-12_employment-contract_startup-name.pdf  
2026-02-14_reference-letter_company-name.pdf  
2026-03-28_linkedin-profile-export.pdf
```
---
## Education

Формат:
```
YYYY-MM-DD_institution_subject_document.ext
```
Примеры:
```
2020-07-15_ngtu_master-diploma.pdf  
2024-06-01_course_vivado-tcl_certificate.pdf  
2023-12-20_university_transcript.pdf
```
---
## Books

Для книг дата обычно не нужна.  
Лучше такой формат:
```
author_title_optional-topic.ext
```
Примеры:
```
harris_digital-design-and-computer-architecture.pdf  
weste_cmos-vlsi-design.pdf  
patterson_computer-organization-and-design.pdf
```
Если авторов несколько:
```
harris-harris_digital-design-and-computer-architecture.pdf
```
Если автора нет или это документация:
```
xilinx_ug903_using-constraints.pdf  
amd_axi-reference-guide.pdf
```
---
## Knowledge

Здесь лучше использовать 2 шаблона.

### Для статей / PDF / материалов:
```
YYYY-MM-DD_topic_source.ext
```
Примеры:
```
2026-03-02_pcie-overview_intel.pdf  
2026-03-10_ddr4-training_material.pdf  
2026-03-14_timing-closure_notes.pdf
```
### Для сохранённых веб-материалов:
```
YYYY-MM-DD_topic_site-name.pdf
```
Примеры:
```
2026-03-18_cdc-design_verilogpro.pdf  
2026-03-21_axi-handshake_zipcpu.pdf
```
---
## Projects_Artifacts

Здесь лучше всего работает структура:
```
YYYY-MM-DD_project_artifact-description.ext
```
Примеры:
```
2026-02-10_artix-board_bringup-report.pdf  
2026-02-14_artix-board_timing-summary.pdf  
2026-02-16_artix-board_bitstream_v1.bit  
2026-02-20_rk3568-server_render-top.png  
2026-02-28_rk3568-server_bom-export.xlsx
```
---

## 1. Общий формат

```
YYYY-MM-DD_type_subject.ext
```
---
## 2. Дата

- формат: `YYYY-MM-DD`
- ставится **в начале**
- используется для:
    - документов
    - финансов
    - медицины
    - работы
    - артефактов проектов

---

## 3. Разделители

- между блоками → `_`
- внутри слов → `-`
```
2026-04-06_tax-return_germany.pdf
```
---
## 4. Регистр и символы

- только `lowercase`
- **без пробелов**
- только латиница

---
## 5. Структура имени

[date]_[type]_[subject]_[optional]

Примеры:
```
2026-04-01_resume_fpga-engineer.pdf  
2026-03-15_invoice_pcbway.pdf  
2026-02-10_artix-board_timing-report.pdf
```
---
## 6. Версии

- только если нужно:
_v1, _v2, _v3

- финальный файл:
без версии

---
## 7. Без мусорных слов

❌ запрещено:
final, new, last, real-final

---
## 8. Книги и справочники

без даты:
author_title.ext

---
## 9. Краткость

- имя = понятно без открытия
- но без «романов»
---
## 10. Один файл — одно место

- не дублировать
- не создавать альтернативных версий без причины