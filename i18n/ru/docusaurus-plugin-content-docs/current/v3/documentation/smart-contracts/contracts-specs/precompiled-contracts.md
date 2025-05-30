# Предварительно скомпилированные контракты

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

*Предварительно скомпилированный смарт-контракт* — это контракт с реализацией C++ в узле.
Когда валидатор запускает транзакцию по такому смарт-контракту, он может выполнить эту реализацию вместо TVM.
Это повышает производительность и позволяет снизить затраты на вычисления.

## Конфигурация

Список предварительно скомпилированных контрактов хранится в конфигурации мастерчейна:

```
precompiled_smc#b0 gas_usage:uint64 = PrecompiledSmc;
precompiled_contracts_config#c0 list:(HashmapE 256 PrecompiledSmc) = PrecompiledContractsConfig;
_ PrecompiledContractsConfig = ConfigParam 45;
```

`list:(HashmapE 256 PrecompiledSmc)` — это карта `(code_hash -> precomplied_smc)`.
Если хэш кода контракта найден в этой карте, то контракт считается *предварительно скомпилированным*.

## Выполнение контракта

Любая транзакция по *предварительно скомпилированному смарт-контракту* (т. е. любому контракту с хэшем кода, найденным в `ConfigParam 45`) выполняется следующим образом:

1. Получите `gas_usage` из конфигурации мастерчейна.
2. Если баланса недостаточно для оплаты газа `gas_usage`, то фаза вычислений завершается неудачей с причиной пропуска `cskip_no_gas`.
3. Код может быть выполнен двумя способами:
4. Если предварительно скомпилированное выполнение отключено или реализация C++ недоступна в текущей версии узла, то TVM работает как обычно. Лимит газа для TVM устанавливается rкак лимит газа транзакции (1M газа).
5. Если предварительно скомпилированная реализация включена и доступна, то выполняется реализация C++.
6. Переопределите [значения фазы вычисления](https://github.com/ton-blockchain/ton/blob/dd5540d69e25f08a1c63760d3afb033208d9c99b/crypto/block/block.tlb#L308): установите `gas_used` на `gas_usage`; установите `vm_steps`, `vm_init_state_hash`, `vm_final_state_hash` на ноль.
7. Плата за вычисления основана на `gas_usage`, а не на фактическом использовании газа TVM.

Когда предварительно скомпилированный контракт выполняется в TVM, 17-й элемент `c7` устанавливается на `gas_usage` и может быть извлечен с помощью инструкции `GETPRECOMPILEDGAS`. Для не предварительно скомпилированных контрактов это значение равно `null`.

Выполнение предварительно скомпилированных контрактов по умолчанию отключено. Запустите `validator-engine` с флагом `--enable-precompiled-smc`, чтобы включить его.

Обратите внимание, что оба способа выполнения предварительно скомпилированного контракта приводят к одной и той же транзакции.
Таким образом, валидаторы с реализацией C++ и без нее могут безопасно сосуществовать в сети.
Это позволяет добавлять новые записи в `ConfigParam 45`, не требуя от всех валидаторов немедленного обновления программного обеспечения узла.

## Доступные реализации

Hic sunt dracones.

## См. также

- [Контракты управления](/v3/documentation/smart-contracts/contracts-specs/governance)
