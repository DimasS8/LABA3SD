# Лабораторная работа №3:

**Задание:**


<img width="1204" height="66" alt="image" src="https://github.com/user-attachments/assets/8fb84af6-ae80-46e7-8105-6648659a4e58" />




# Реализация:

**Листинг программы:**
  ````
import time
import random
from collections import deque
from copy import deepcopy

# --- 1. Реализация структур данных ---

class CoinArray:
    """Реализация на основе массива."""
    def __init__(self, initial_state=None):
        self.coins = initial_state.copy() if initial_state is not None else []
        self.size = len(self.coins)

    def flip(self, index):
        self.coins[index] = 1 - self.coins[index]

    def get_count_up(self):
        return sum(self.coins)

    def copy(self):
        return CoinArray(self.coins.copy())
        
    def to_list(self):
        """Возвращает копию внутреннего состояния в виде списка."""
        return self.coins.copy()

    @classmethod
    def create_random(cls, n_up, n_down):
        state = [1] * n_up + [0] * n_down
        random.shuffle(state)
        return cls(state)


class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

class CoinLinkedList:
    """Реализация на основе связного списка."""
    def __init__(self, initial_state=None):
        self.head = None
        self.size = 0
        if initial_state:
            self.size = len(initial_state)
            self.head = Node(initial_state[0])
            current = self.head
            for value in initial_state[1:]:
                current.next = Node(value)
                current = current.next

    def get_node_at(self, index):
        current = self.head
        for _ in range(index):
            current = current.next
        return current

    def flip(self, index):
        node = self.get_node_at(index)
        node.value = 1 - node.value

    def get_count_up(self):
        count = 0
        current = self.head
        while current:
            count += current.value
            current = current.next
        return count

    def copy(self):
         return CoinLinkedList(self.to_list())
        
    def to_list(self):
        """Преобразует связный список в обычный список Python."""
        out = []
        current = self.head
        while current:
            out.append(current.value)
            current = current.next
        return out

    @classmethod
    def create_random(cls, n_up, n_down):
         array_ds = CoinArray.create_random(n_up, n_down)
         return cls(array_ds.coins)


class CoinDeque:
    """Реализация на основе deque из стандартной библиотеки."""
    def __init__(self, initial_state=None):
         self.coins = deque(initial_state) if initial_state is not None else deque()
         self.size = len(self.coins)

    def flip(self, index):
         # Доступ по индексу в deque работает за O(1)
         self.coins[index] = 1 - self.coins[index]

    def get_count_up(self):
         return sum(self.coins)

    def copy(self):
         new_instance = CoinDeque()
         new_instance.coins = self.coins.copy()
         new_instance.size = self.size
         return new_instance
        
    def to_list(self):
        """Преобразует deque в обычный список Python."""
        return list(self.coins)

    @classmethod
    def create_random(cls, n_up, n_down):
         array_ds = CoinArray.create_random(n_up, n_down)
         return cls(array_ds.coins)


# --- 2. Логика симуляции и генетического алгоритма ---

def simulate(coins_ds, S, K):
    """
    Симулирует K ходов переворота монет.
    :param coins_ds: Экземпляр структуры данных (CoinArray/LinkedList/Deque).
    :param S: Шаг переворота.
    :param K: Количество ходов.
    :return: Количество монет гербом вверх после симуляции.
    """
    current_pos = 0
    
    for _ in range(K):
        flip_pos = (current_pos + S - 1) % coins_ds.size
        coins_ds.flip(flip_pos)
        
        # Следующий отсчет начинается с перевернутой монеты
        current_pos = flip_pos
    
    return coins_ds.get_count_up()


def genetic_algorithm(N, M, S, K, L, data_structure_class,
                      pop_size=50, generations=200, mutation_rate=0.05):
    """
    Генетический алгоритм для поиска расстановки.
    :param data_structure_class: Класс структуры данных (CoinArray и т.д.).
    :return: Найденная расстановка (в виде списка) или None.
    """
    print(f"\n--- Запуск генетического алгоритма для {data_structure_class.__name__} ---")
    
    # 1. Создание начальной популяции
    start_time_pop = time.perf_counter()
    population = [data_structure_class.create_random(N, M) for _ in range(pop_size)]
    pop_gen_time = time.perf_counter() - start_time_pop

    for generation in range(generations):
         # Оценка приспособленности (fitness) каждой особи
         fitness_scores = []
         for individual in population:
             # Создаем копию для симуляции, чтобы не портить исходную особь
             sim_copy = individual.copy()
             final_count_up = simulate(sim_copy, S, K)
             # Чем ближе к L, тем выше приспособленность. Используем обратное расстояние.
             fitness = 1.0 / (abs(final_count_up - L) + 1)
             fitness_scores.append((fitness, individual))
         
         # Сортировка по убыванию приспособленности
         fitness_scores.sort(key=lambda x: x[0], reverse=True)
         
         # Проверка на решение
         best_fitness, best_individual = fitness_scores[0]
         best_count_up = simulate(best_individual.copy(), S, K)
         if best_count_up == L:
             print(f"Решение найдено в поколении {generation}!")
             return best_individual.to_list() if hasattr(best_individual, 'to_list') else best_individual.coins

         # 2. Отбор (селекция) - берем топ-20%
         elite_size = int(pop_size * 0.2)
         next_generation = [ind for _, ind in fitness_scores[:elite_size]]
         
         # 3. Скрещивание (кроссинговер) и мутация для пополнения популяции
         while len(next_generation) < pop_size:
             # Выбор родителей (турнирная селекция)
             parent1 = random.choice(fitness_scores[:pop_size//2])[1]
             parent2 = random.choice(fitness_scores[:pop_size//2])[1]
             
             # Скрещивание: берем часть от одного родителя и часть от другого
             split_point = random.randint(1, N+M-1)
             
             p1_state = parent1.to_list() if hasattr(parent1, 'to_list') else parent1.coins.copy()
             p2_state = parent2.to_list() if hasattr(parent2, 'to_list') else parent2.coins.copy()
             
             child_state = p1_state[:split_point] + p2_state[split_point:]
             
             # Мутация с вероятностью mutation_rate: меняем местами две случайные монеты
             if random.random() < mutation_rate:
                 i, j = random.sample(range(N+M), 2)
                 child_state[i], child_state[j] = child_state[j], child_state[i]
             
             child_ds = data_structure_class(child_state)
             next_generation.append(child_ds)
         
         population = next_generation

    print("Алгоритм завершен. Решение не найдено. Возвращаем лучший результат.")
    best_fitness, best_individual = fitness_scores[0]
    best_count_up = simulate(best_individual.copy(), S, K)
    print(f"Лучший результат: {best_count_up} монет вверх из требуемых {L}.")
    return best_individual.to_list() if hasattr(best_individual, 'to_list') else best_individual.coins


# --- 3. Основная программа и сравнение ---
if __name__ == '__main__':
     # Входные данные задачи (небольшие для быстрой демонстрации)
     N = 5       # Монет гербом вверх
     M = 5       # Монет гербом вниз
     S = 3       # Переворачиваем каждую S-ую монету
     K = 50      # Количество ходов
     L = 4       # Требуемое кол-во "вверх" в конце

     print(f"Задача: Найти расстановку {N} гербом вверх и {M} вниз,\n"
           f"чтобы после {K} переворотов каждой {S}-й монеты\n"
           f"вверх оказалось ровно {L} монет.")
     
     structures_to_test = [CoinArray, CoinDeque]
     
     try:
         # Попытка запустить и для связного списка (скорее всего, будет очень долго)
         structures_to_test.append(CoinLinkedList)
     except RecursionError:
         print("\nПредупреждение: Для больших данных связный список может вызвать переполнение стека.")
     
     results = {}
     
     for struct_class in structures_to_test:
         try:
             start_total_time = time.perf_counter()
             
             solution_state = genetic_algorithm(N, M, S, K, L, struct_class,
                                               pop_size=30, generations=50)
             
             total_time_taken = time.perf_counter() - start_total_time
             
             if solution_state is not None:
                 print(f"\n--- Результат для {struct_class.__name__} ---")
                 print(f"Найденная расстановка: {solution_state}")
                 
                 # Финальная проверка решения с помощью массива (самый быстрый способ проверки)
                 final_check_count = simulate(CoinArray(solution_state), S, K)
                 print(f"ПРОВЕРКА: После {K} ходов гербом вверх лежит {final_check_count} монет.")
                 print(f"Время работы алгоритма: {total_time_taken:.3f} секунд")
                 
                 results[struct_class.__name__] = {
                     'state': solution_state,
                     'time': total_time_taken,
                     'final_check': final_check_count
                 }
                 
         except Exception as e:
             print(f"\nОшибка при работе с {struct_class.__name__}: {e}")
print('Манжин Дмитрий Евгеньевич, группа 090301-ПОВа-025')
````
# Результат выполнения программы:



