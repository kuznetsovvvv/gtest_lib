## Laboratory work V

# Homework
1. Создайте CMakeList.txt для библиотеки banking.
2. Создайте модульные тесты на классы Transaction и Account.
-Используйте mock-объекты.
-Покрытие кода должно составлять 100%.
3. Настройте сборочную процедуру на TravisCI.
4. Настройте Coveralls.io.

# 1)Для начала клонируем репозиторий лабораторной https://github.com/tp-labs/lab05
```bash
Команда: git clone https://github.com/tp-labs/lab05
```
Далее переходим в репозиторий и создаем файл для библиотеки banking CMakeListst.txt
Записываем в файл:

```bash
cmake_minimum_required(VERSION 3.4)
project(Test_banking)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)

if(BUILD_TESTS)
  add_compile_options(--coverage)
endif()

add_library(banking STATIC ${CMAKE_CURRENT_SOURCE_DIR}/banking/Transaction.cpp ${CMAKE_CURRENT_SOURCE_DIR}/banking/Account.cpp)
target_include_directories(banking PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/banking)
target_link_libraries(banking gcov)

if(BUILD_TESTS)
  enable_testing()
  add_subdirectory(third-party/gtest)
  file(GLOB BANKING_TEST_SOURCES tests/*.cpp)
  add_executable(check ${BANKING_TEST_SOURCES})
  target_link_libraries(check banking gtest_main gmock_main)
  add_test(NAME check COMMAND check)
endif()
```
Линкуем gcov к библиотеке banking, - чтобы были отчёты о покрытии 

# 2) Создаем папку tests и создаем в ней файл tests.cpp

```bash
Команда:mkdir tests
Команда:touch tests.cpp
```

Пишем тесты в этот файл:

```bash
#include <gtest/gtest.h>
#include <gmock/gmock.h>

#include "Account.h"
#include "Transaction.h"

class AccountMock : public Account {
public:
	AccountMock(int id, int balance) : Account(id, balance) {}
	MOCK_CONST_METHOD0(GetBalance, int());
	MOCK_METHOD1(ChangeBalance, void(int diff));
	MOCK_METHOD0(Lock, void());
	MOCK_METHOD0(Unlock, void());
};
class TransactionMock : public Transaction {
public:
	MOCK_METHOD3(Make, bool(Account& from, Account& to, int sum));
};

TEST(Account, Mock) {
	AccountMock acc(1, 100);
	EXPECT_CALL(acc, GetBalance()).Times(1);
	EXPECT_CALL(acc, ChangeBalance(testing::_)).Times(2);
	EXPECT_CALL(acc, Lock()).Times(2);
	EXPECT_CALL(acc, Unlock()).Times(1);
	acc.GetBalance();
	acc.ChangeBalance(100); // throw
	acc.Lock();
	acc.ChangeBalance(100);
	acc.Lock(); // throw
	acc.Unlock();
}

TEST(Account, SimpleTest) {
	Account acc(1, 100);
	EXPECT_EQ(acc.id(), 1);
	EXPECT_EQ(acc.GetBalance(), 100);
	EXPECT_THROW(acc.ChangeBalance(100), std::runtime_error);
	EXPECT_NO_THROW(acc.Lock());
	acc.ChangeBalance(100);
	EXPECT_EQ(acc.GetBalance(), 200);
	EXPECT_THROW(acc.Lock(), std::runtime_error);
}

TEST(Transaction, Mock) {
	TransactionMock tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	EXPECT_CALL(tr, Make(testing::_, testing::_, testing::_))
	.Times(6);
	tr.set_fee(100);
	tr.Make(ac1, ac2, 199);
	tr.Make(ac2, ac1, 500);
	tr.Make(ac2, ac1, 300);
	tr.Make(ac1, ac1, 0); // throw
	tr.Make(ac1, ac2, -1); // throw
	tr.Make(ac1, ac2, 99); // throw
}

TEST(Transaction, SimpleTest) {
	Transaction tr;
	Account ac1(1, 50);
	Account ac2(2, 500);
	tr.set_fee(100);
	EXPECT_EQ(tr.fee(), 100);
	EXPECT_THROW(tr.Make(ac1, ac1, 0), std::logic_error);
	EXPECT_THROW(tr.Make(ac1, ac2, -1), std::invalid_argument);
	EXPECT_THROW(tr.Make(ac1, ac2, 99), std::logic_error);
	EXPECT_FALSE(tr.Make(ac1, ac2, 199));
	EXPECT_FALSE(tr.Make(ac2, ac1, 500));
	EXPECT_TRUE(tr.Make(ac2, ac1, 300));
}
```

---

Также в процессе мы должны добавить gtest в папку third-party, как указано в туториале к лабораторной работе 
```bash
Команда:git submodule add https://github.com/google/googletest third-party/gtest
Команда:cd third-party/gtest && git checkout release-1.8.1 && cd ../..
```

---

# 3) Создаем сборку для Github Actions 
Создаем директорию .github/workflows и файл CI.yml:

```bash
Команда:mkdir .github/workflows
Команда:touch CI.yml
```

С помощью текстового редактора nano записываем в файл CI.yml:

```bash
name: CMake

on:
  push:
    branches: [master]
  pull_request:
    branches: [master]

jobs:
  build_Linux:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Adding gtest
      run: git clone https://github.com/google/googletest.git third-party/gtest -b release-1.11.0

    - name: Install lcov
      run: sudo apt-get install -y lcov

    - name: Config banking with tests
      run: cmake -H. -B ${{github.workspace}}/build -DBUILD_TESTS=ON -DCMAKE_CXX_FLAGS='--coverage'

    - name: Build banking
      run: cmake --build ${{github.workspace}}/build

    - name: Run tests
      run: |
        build/check
        lcov --directory . --capture --output-file coverage.info
        lcov --remove coverage.info '/usr/*' --output-file coverage.info
        lcov --remove coverage.info '${{github.workspace}}/third-party/gtest/*' --output-file coverage.info
        lcov --list coverage.info

    - name: Coveralls
      uses: coverallsapp/github-action@master
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        path-to-lcov: ${{ github.workspace }}/coverage.info

```
При запуске тестов на Github Actions обнаружилась ошибка в коде класса: Transaction::Make(Account& from, Account& to, int sum). В функции Credit(to, sum) передается неправильный аргумент, должно быть Credit(from, sum) чтобы зачислить средства на счет отправителя, а не на счет получателя.


# 4) Регистрируемся на сайте https://coveralls.io с помощью Github Account 
Подсоединяем нужный репозиторий через графический интерфейс coveralls.io
Копируем с разметкой Markdown для отправки информации о покрытии кода, содержащейся в файле coverage.info, на сервис Coveralls.io.

Результат:
[![Coverage Status](https://coveralls.io/repos/github/kuznetsovvvv/lab005/badge.svg?branch=master)](https://coveralls.io/github/kuznetsovvvv/lab005?branch=master)





