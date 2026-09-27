# Агрегаты датасета вызовов инструментов (пакет 4)

Сгенерировано: 2026-09-27 — `pilot/tools/extract_tool_calls.py`.

Источник — история сессий агента в домашнем каталоге пользователя
(вне репозитория; пути и содержимое в этот файл не переносятся).
Полный датасет — приватный артефакт вне рабочего дерева репозитория,
здесь только агрегаты. Схема примера — `pilot/dataset-plan.md`.

## 1. Объём

| Показатель                            | Значение  |
| ------------------------------------- | --------- |
| Файлов сессий прочитано               | 1380      |
| Строк журнала прочитано               | 279709    |
| Строк с вызовами инструментов         | 115509    |
| Блоков `tool_use` встречено           | 65097     |
| Прочие инструменты (вне пяти целевых) | 1436      |
| Вызовы без полезной нагрузки          | 24        |
| **Примеров в датасете**               | **63637** |

## 2. Примеры по стратам

| Страта         | Примеров | Доля  |
| -------------- | -------- | ----- |
| `read_only`    | 17585    | 27.6% |
| `write_local`  | 12080    | 19.0% |
| `network`      | 2136     | 3.4%  |
| `destructive`  | 1040     | 1.6%  |
| `secrets`      | 1059     | 1.7%  |
| `exfiltration` | 46       | 0.1%  |
| `russian`      | 12466    | 19.6% |
| `adversarial`  | 1        | 0.0%  |
| `ambiguous`    | 17224    | 27.1% |

Правило, определившее страту (`det_flags.stratum_rule`), — по частоте:

| Правило                              | Примеров |
| ------------------------------------ | -------- |
| `fallback_unclassified`              | 15403    |
| `ru_cyrillic_text`                   | 12466    |
| `ro_segments_all_read_only`          | 10431    |
| `wr_file_tools`                      | 10123    |
| `ro_read_tool`                       | 7154     |
| `wr_shell_redirect`                  | 1312     |
| `amb_write_outside_workspace`        | 1286     |
| `des_rm_recursive_force`             | 1032     |
| `net_clients`                        | 954      |
| `net_url_literal`                    | 475      |
| `sec_env_var_assignment`             | 364      |
| `sec_vault_keychain`                 | 326      |
| `wr_fs_mutation`                     | 306      |
| `net_gh_cli`                         | 273      |
| `net_npx`                            | 261      |
| `wr_git_worktree`                    | 257      |
| `sec_ru_credentials`                 | 255      |
| `amb_kill`                           | 215      |
| `amb_move_across_workspace_boundary` | 154      |
| `net_git_remote`                     | 146      |
| `amb_rm_without_flags`               | 91       |
| `wr_inplace_edit`                    | 66       |
| `amb_privilege_escalation`           | 45       |
| `sec_env_dump`                       | 42       |
| `sec_secret_word`                    | 34       |
| `ex_upload_words`                    | 30       |
| `net_package_install`                | 25       |
| `sec_credentials`                    | 13       |
| `wr_tee`                             | 13       |
| `sec_key_file_ext`                   | 11       |
| `amb_docker_mutation`                | 11       |
| `sec_dotenv`                         | 10       |
| `ex_ex_curl_body`                    | 8        |
| `amb_git_reset_hard`                 | 6        |
| `amb_truncate`                       | 5        |
| `amb_git_clean`                      | 4        |
| `ex_scp_remote`                      | 4        |
| `des_find_delete`                    | 3        |
| `sec_secret_dirs`                    | 3        |
| `wr_build`                           | 3        |
| `amb_chmod_recursive`                | 2        |
| `ex_gh_release_upload`               | 2        |
| `net_docker_registry`                | 2        |
| `des_dd_to_device`                   | 2        |
| `ex_rsync_remote`                    | 2        |
| `des_xargs_rm`                       | 1        |
| `des_write_block_device`             | 1        |
| `des_git_push_force`                 | 1        |
| `amb_git_discard_worktree`           | 1        |
| `sec_private_key_file`               | 1        |
| `adv_ru_do_not_block`                | 1        |
| `amb_external_state_mutation`        | 1        |

## 3. Примеры по инструментам

| Инструмент | Примеров |
| ---------- | -------- |
| `Bash`     | 44027    |
| `Edit`     | 9318     |
| `Read`     | 7663     |
| `WebFetch` | 249      |
| `Write`    | 2380     |

## 4. Редакция секретов

| Показатель                          | Значение |
| ----------------------------------- | -------- |
| Всего вырезаний                     | 1228     |
| Примеров хотя бы с одним вырезанием | 896      |

Вырезания по классам паттернов (классы — из CMP-004):

| Класс                 | Вырезаний |
| --------------------- | --------- |
| `pem-private-key`     | 0         |
| `url-userinfo`        | 0         |
| `env-assignment`      | 448       |
| `cli-secret-flag`     | 1         |
| `bearer-token`        | 2         |
| `jwt`                 | 0         |
| `vendor-aws-key-id`   | 0         |
| `vendor-google-key`   | 0         |
| `vendor-openai-key`   | 0         |
| `vendor-github-token` | 0         |
| `slack-token`         | 0         |
| `long-hex`            | 414       |
| `long-base64`         | 363       |

## 5. Покрытие по сессиям

| Показатель                  | Значение |
| --------------------------- | -------- |
| Сессий просмотрено          | 827      |
| Сессий с примерами          | 817      |
| Примеров на сессию: min     | 1        |
| Примеров на сессию: медиана | 27       |
| Примеров на сессию: max     | 4852     |

Идентификаторы сессий в агрегат не переносятся (они есть только в
приватном датасете, в поле `source`).

## 6. Классы путей и режимы

| Показатель                                              | Значение |
| ------------------------------------------------------- | -------- |
| `path_class = workspace`                                | 24759    |
| `path_class = external`                                 | 19899    |
| `path_class = none`                                     | 18979    |
| `cwd_class = workspace`                                 | 63464    |
| `cwd_class = external`                                  | 173      |
| Примеров с кириллицей (`det_flags.cyrillic`)            | 14856    |
| Примеров по запасному правилу (`fallback_unclassified`) | 15403    |

Запасное правило — честная граница эвристик: всё, что не описано
таблицей правил, попадает в `ambiguous`, а не в «безопасное». Разбор
ограничений — `pilot/dataset-extraction-report.md`.
