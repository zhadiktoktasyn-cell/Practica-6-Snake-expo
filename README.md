
import React, { useCallback, useEffect, useState } from 'react';
import {
  Alert,
  ActivityIndicator,
  FlatList,
  KeyboardAvoidingView,
  Platform,
  SafeAreaView,
  ScrollView,
  StatusBar,
  StyleSheet,
  Text,
  TextInput,
  TouchableOpacity,
  View,
} from 'react-native';
import {
  SQLiteProvider,
  useSQLiteContext,
} from 'expo-sqlite';

const C = {
  bg: '#0b1017',
  card: '#151e29',
  green: '#b7f34a',
  text: '#f4f7fb',
  muted: '#91a0b2',
  border: '#273444',
  red: '#ff6565',
};

const initialGenres = [
  'Экшен',
  'RPG',
  'Шутер',
  'Стратегия',
  'Приключения',
  'Гонки',
  'Симулятор',
  'Хоррор',
  'Спорт',
  'Платформер',
];

async function initializeDatabase(db) {
  await db.execAsync('PRAGMA foreign_keys = ON;');

  await db.execAsync(`
    CREATE TABLE IF NOT EXISTS Genres (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL UNIQUE
    );

    CREATE TABLE IF NOT EXISTS Games (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT NOT NULL DEFAULT '',
      created_at TEXT NOT NULL,
      genre_id INTEGER NOT NULL,
      release_year INTEGER NOT NULL,
      platform TEXT NOT NULL,
      developer TEXT NOT NULL,
      rating REAL NOT NULL DEFAULT 0,
      FOREIGN KEY (genre_id) REFERENCES Genres(id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT
    );
  `);

  for (const name of initialGenres) {
    await db.runAsync(
      'INSERT OR IGNORE INTO Genres (name) VALUES (?)',
      name
    );
  }

  const count = await db.getFirstAsync(
    'SELECT COUNT(*) AS count FROM Games'
  );

  if (count.count === 0) {
    const genre = await db.getFirstAsync(
      'SELECT id FROM Genres WHERE name = ?',
      'Экшен'
    );

    await db.runAsync(
      `INSERT INTO Games
       (title, description, created_at, genre_id,
        release_year, platform, developer, rating)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
      'Cyberpunk 2077',
      'Ролевая игра с открытым миром в футуристическом Найт-Сити.',
      new Date().toISOString(),
      genre.id,
      2020,
      'PC, PlayStation, Xbox',
      'CD Projekt Red',
      8.5
    );
  }
}

function Button({ title, onPress, secondary, danger, disabled }) {
  return (
    <TouchableOpacity
      onPress={onPress}
      disabled={disabled}
      style={[
        styles.button,
        secondary && styles.secondaryButton,
        danger && styles.dangerButton,
        disabled && { opacity: 0.5 },
      ]}
    >
      <Text
        style={[
          styles.buttonText,
          secondary && { color: C.green },
          danger && { color: '#fff' },
        ]}
      >
        {title}
      </Text>
    </TouchableOpacity>
  );
}

function AppContent() {
  const db = useSQLiteContext();

  const [games, setGames] = useState([]);
  const [genres, setGenres] = useState([]);
  const [search, setSearch] = useState('');
  const [genreFilter, setGenreFilter] = useState(null);
  const [sort, setSort] = useState('title');
  const [screen, setScreen] = useState('list');
  const [selected, setSelected] = useState(null);
  const [form, setForm] = useState(null);
  const [busy, setBusy] = useState(true);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState('');

  const loadGenres = useCallback(async () => {
    const rows = await db.getAllAsync(
      'SELECT id, name FROM Genres ORDER BY name'
    );
    setGenres(rows);
  }, [db]);

  const loadGames = useCallback(async () => {
    setBusy(true);
    setError('');

    try {
      let sql = `
        SELECT Games.*, Genres.name AS genre_name
        FROM Games
        JOIN Genres ON Games.genre_id = Genres.id
        WHERE Games.title LIKE ?
      `;
      const params = [`%${search.trim()}%`];

      if (genreFilter !== null) {
        sql += ' AND Games.genre_id = ?';
        params.push(genreFilter);
      }

      const allowedSorts = {
        title: 'Games.title COLLATE NOCASE ASC',
        year: 'Games.release_year DESC',
        rating: 'Games.rating DESC',
      };

      sql += ` ORDER BY ${allowedSorts[sort] || allowedSorts.title}`;

      const rows = await db.getAllAsync(sql, ...params);
      setGames(rows);
    } catch (e) {
      setError('Не удалось загрузить игры из базы данных.');
    } finally {
      setBusy(false);
    }
  }, [db, search, genreFilter, sort]);

  useEffect(() => {
    loadGenres().catch(() => setError('Ошибка загрузки жанров.'));
  }, [loadGenres]);

  useEffect(() => {
    loadGames();
  }, [loadGames]);

  function updateForm(key, value) {
    setForm(previous => ({ ...previous, [key]: value }));
  }

  function openAddForm() {
    setForm({
      title: '',
      description: '',
      genre_id: genres[0]?.id ?? null,
      release_year: String(new Date().getFullYear()),
      platform: '',
      developer: '',
      rating: '0',
    });
    setScreen('form');
  }

  function openEditForm(game) {
    setForm({
      id: game.id,
      title: game.title,
      description: game.description,
      genre_id: game.genre_id,
      release_year: String(game.release_year),
      platform: game.platform,
      developer: game.developer,
      rating: String(game.rating),
    });
    setScreen('form');
  }

  async function saveGame() {
    if (!form.title.trim()) {
      Alert.alert('Ошибка', 'Введите название игры.');
      return;
    }

    const year = Number(form.release_year);
    const rating = Number(form.rating);

    if (!Number.isInteger(year) || year < 1950 || year > 2100) {
      Alert.alert('Ошибка', 'Укажите корректный год выпуска.');
      return;
    }

    if (
      !Number.isFinite(rating) ||
      rating < 0 ||
      rating > 10
    ) {
      Alert.alert('Ошибка', 'Рейтинг должен быть от 0 до 10.');
      return;
    }

    if (!form.genre_id || !form.platform.trim() || !form.developer.trim()) {
      Alert.alert(
        'Ошибка',
        'Выберите жанр и заполните платформу и разработчика.'
      );
      return;
    }

    setSaving(true);

    try {
      const values = [
        form.title.trim(),
        form.description.trim(),
        form.genre_id,
        year,
        form.platform.trim(),
        form.developer.trim(),
        rating,
      ];

      if (form.id) {
        await db.runAsync(
          `UPDATE Games
           SET title = ?, description = ?, genre_id = ?,
               release_year = ?, platform = ?, developer = ?,
               rating = ?
           WHERE id = ?`,
          ...values,
          form.id
        );
      } else {
        await db.runAsync(
          `INSERT INTO Games
           (title, description, created_at, genre_id,
            release_year, platform, developer, rating)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
          values[0],
          values[1],
          new Date().toISOString(),
          ...values.slice(2)
        );
      }

      setScreen('list');
      await loadGames();
      await loadGenres();
      Alert.alert('Готово', form.id ? 'Игра обновлена.' : 'Игра добавлена.');
    } catch (e) {
      Alert.alert('Ошибка', 'Не удалось сохранить игру.');
    } finally {
      setSaving(false);
    }
  }

  function deleteGame(game) {
    Alert.alert(
      'Удалить игру?',
      `Удалить «${game.title}» из каталога?`,
      [
        { text: 'Отмена', style: 'cancel' },
        {
          text: 'Удалить',
          style: 'destructive',
          onPress: async () => {
            try {
              await db.runAsync(
                'DELETE FROM Games WHERE id = ?',
                game.id
              );
              setSelected(null);
              setScreen('list');
              await loadGames();
            } catch (e) {
              Alert.alert('Ошибка', 'Не удалось удалить игру.');
            }
          },
        },
      ]
    );
  }

  async function addGenre() {
    Alert.prompt(
      'Новый жанр',
      'Введите название жанра:',
      async name => {
        const value = name?.trim();
        if (!value) return;

        try {
          await db.runAsync(
            'INSERT INTO Genres (name) VALUES (?)',
            value
          );
          await loadGenres();
          Alert.alert('Готово', 'Жанр добавлен.');
        } catch (e) {
          Alert.alert('Ошибка', 'Такой жанр уже существует.');
        }
      }
    );
  }

  function openDetails(game) {
    setSelected(game);
    setScreen('detail');
  }

  function renderGame({ item }) {
    return (
      <TouchableOpacity
        style={styles.gameCard}
        onPress={() => openDetails(item)}
        activeOpacity={0.8}
      >
        <View style={styles.gameIcon}>
          <Text style={styles.gameEmoji}>🎮</Text>
        </View>
        <View style={styles.gameInfo}>
          <Text style={styles.gameTitle}>{item.title}</Text>
          <Text style={styles.muted}>
            {item.genre_name} · {item.release_year}
          </Text>
          <Text style={styles.muted} numberOfLines={1}>
            {item.platform}
          </Text>
          <Text style={styles.rating}>★ {item.rating.toFixed(1)} / 10</Text>
        </View>
        <Text style={styles.arrow}>›</Text>
      </TouchableOpacity>
    );
  }

  function renderList() {
    return (
      <>
        <View style={styles.header}>
          <Text style={styles.logo}>GAME<Text style={{ color: C.green }}>VAULT</Text></Text>
          <Text style={styles.subtitle}>ТВОЯ КОЛЛЕКЦИЯ ВИДЕОИГР</Text>
        </View>

        <View style={styles.toolbar}>
          <TextInput
            value={search}
            onChangeText={setSearch}
            placeholder="Поиск игры..."
            placeholderTextColor={C.muted}
            style={styles.search}
          />
          <TouchableOpacity
            style={styles.addButton}
            onPress={openAddForm}
          >
            <Text style={styles.addButtonText}>＋</Text>
          </TouchableOpacity>
        </View>

        <ScrollView
          horizontal
          showsHorizontalScrollIndicator={false}
          style={styles.filterScroll}
          contentContainerStyle={styles.filters}
        >
          <TouchableOpacity
            onPress={() => setGenreFilter(null)}
            style={[styles.chip, genreFilter === null && styles.activeChip]}
          >
            <Text style={[styles.chipText, genreFilter === null && styles.activeChipText]}>
              Все жанры
            </Text>
          </TouchableOpacity>

          {genres.map(g => (
            <TouchableOpacity
              key={g.id}
              onPress={() => setGenreFilter(g.id)}
              style={[styles.chip, genreFilter === g.id && styles.activeChip]}
            >
              <Text style={[styles.chipText, genreFilter === g.id && styles.activeChipText]}>
                {g.name}
              </Text>
            </TouchableOpacity>
          ))}
        </ScrollView>

        <View style={styles.listHeading}>
          <Text style={styles.sectionTitle}>Мои игры</Text>
          <Text style={styles.muted}>{games.length} записей</Text>
        </View>

        <View style={styles.sortRow}>
          {[
            ['title', 'По названию'],
            ['year', 'По году'],
            ['rating', 'По рейтингу'],
          ].map(([key, label]) => (
            <TouchableOpacity
              key={key}
              onPress={() => setSort(key)}
              style={[styles.sortChip, sort === key && styles.sortActive]}
            >
              <Text style={styles.sortText}>{label}</Text>
            </TouchableOpacity>
          ))}
        </View>

        {busy ? (
          <View style={styles.center}>
            <ActivityIndicator color={C.green} size="large" />
            <Text style={styles.muted}>Загрузка базы данных...</Text>
          </View>
        ) : error ? (
          <View style={styles.center}>
            <Text style={styles.error}>{error}</Text>
            <Button title="Повторить" onPress={loadGames} />
          </View>
        ) : (
          <FlatList
            data={games}
            keyExtractor={item => String(item.id)}
            renderItem={renderGame}
            contentContainerStyle={styles.list}
            keyboardShouldPersistTaps="handled"
            ListEmptyComponent={
              <Text style={styles.empty}>
                Игр пока нет. Добавь первую игру!
              </Text>
            }
          />
        )}

        <View style={styles.bottom}>
          <TouchableOpacity onPress={addGenre}>
            <Text style={styles.bottomLink}>＋ Управление жанрами</Text>
          </TouchableOpacity>
          <Text style={styles.bottomCaption}>Локальная база данных SQLite</Text>
        </View>
      </>
    );
  }

  function renderField(label, key, placeholder, multiline = false) {
    return (
      <View style={styles.field} key={key}>
        <Text style={styles.label}>{label}</Text>
        <TextInput
          style={[styles.input, multiline && styles.multiline]}
          value={form[key]}
          onChangeText={value => updateForm(key, value)}
          placeholder={placeholder}
          placeholderTextColor={C.muted}
          multiline={multiline}
          keyboardType={
            key === 'release_year' || key === 'rating'
              ? 'numeric'
              : 'default'
          }
        />
      </View>
    );
  }

  function renderForm() {
    return (
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      >
        <ScrollView
          keyboardShouldPersistTaps="handled"
          contentContainerStyle={styles.formContent}
        >
          <Button
            title="← Назад"
            secondary
            onPress={() => setScreen(selected ? 'detail' : 'list')}
          />
          <Text style={styles.pageTitle}>
            {form.id ? 'Редактировать игру' : 'Новая игра'}
          </Text>
          <Text style={styles.muted}>Заполни информацию об игре</Text>

          {renderField('Название *', 'title', 'Например, Elden Ring')}
          {renderField('Описание', 'description', 'Расскажи об игре...', true)}

          <Text style={styles.label}>Жанр *</Text>
          <View style={styles.genreOptions}>
            {genres.map(g => (
              <TouchableOpacity
                key={g.id}
                onPress={() => updateForm('genre_id', g.id)}
                style={[
                  styles.chip,
                  form.genre_id === g.id && styles.activeChip,
                ]}
              >
                <Text
                  style={[
                    styles.chipText,
                    form.genre_id === g.id && styles.activeChipText,
                  ]}
                >
                  {g.name}
                </Text>
              </TouchableOpacity>
            ))}
          </View>

          {renderField('Год выпуска *', 'release_year', '2022')}
          {renderField('Платформа *', 'platform', 'PC, PlayStation 5')}
          {renderField('Разработчик *', 'developer', 'Название студии')}
          {renderField('Рейтинг (0–10)', 'rating', '8.5')}

          <Button
            title={saving ? 'Сохранение...' : 'Сохранить игру'}
            onPress={saveGame}
            disabled={saving}
          />
          <Button
            title="Отмена"
            secondary
            onPress={() => setScreen(selected ? 'detail' : 'list')}
          />
        </ScrollView>
      </KeyboardAvoidingView>
    );
  }

  function renderDetail() {
    if (!selected) return null;

    return (
      <ScrollView contentContainerStyle={styles.formContent}>
        <Button
          title="← К каталогу"
          secondary
          onPress={() => {
            setSelected(null);
            setScreen('list');
          }}
        />

        <View style={styles.detailHero}>
          <Text style={{ fontSize: 70 }}>🎮</Text>
          <Text style={styles.detailTitle}>{selected.title}</Text>
          <Text style={styles.rating}>★ {selected.rating.toFixed(1)} / 10</Text>
          <Text style={styles.muted}>
            {selected.genre_name} · {selected.release_year}
          </Text>
        </View>

        <Text style={styles.sectionTitle}>Информация об игре</Text>

        {[
          ['Жанр', selected.genre_name],
          ['Год выпуска', selected.release_year],
          ['Платформа', selected.platform],
          ['Разработчик', selected.developer],
          ['Дата добавления', selected.created_at.slice(0, 10)],
          ['Описание', selected.description || 'Описание отсутствует'],
        ].map(([label, value]) => (
          <View style={styles.detailRow} key={label}>
            <Text style={styles.label}>{label}</Text>
            <Text style={styles.detailValue}>{String(value)}</Text>
          </View>
        ))}

        <Button
          title="✎ Редактировать"
          onPress={() => openEditForm(selected)}
        />
        <Button
          title="Удалить игру"
          danger
          onPress={() => deleteGame(selected)}
        />
      </ScrollView>
    );
  }

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="light-content" backgroundColor={C.bg} />
      {screen === 'list'
        ? renderList()
        : screen === 'form'
        ? renderForm()
        : renderDetail()}
    </SafeAreaView>
  );
}

export default function App() {
  return (
    <SQLiteProvider databaseName="gamevault.db" onInit={initializeDatabase}>
      <AppContent />
    </SQLiteProvider>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: C.bg },
  header: { paddingHorizontal: 20, paddingTop: 20, paddingBottom: 18 },
  logo: { fontSize: 29, fontWeight: '900', color: C.text, letterSpacing: 1 },
  subtitle: { color: C.muted, fontSize: 10, letterSpacing: 2, marginTop: 5 },
  toolbar: { flexDirection: 'row', gap: 10, paddingHorizontal: 16 },
  search: {
    flex: 1, backgroundColor: C.card, borderColor: C.border,
    borderWidth: 1, borderRadius: 12, paddingHorizontal: 14,
    color: C.text, height: 46,
  },
  addButton: {
    width: 48, backgroundColor: C.green, borderRadius: 12,
    justifyContent: 'center', alignItems: 'center',
  },
  addButtonText: { fontSize: 28, color: C.bg, fontWeight: '700' },
  filterScroll: { maxHeight: 52, marginTop: 12, flexGrow: 0 },
  filters: { paddingHorizontal: 16, gap: 8, alignItems: 'center' },
  chip: {
    backgroundColor: C.card, borderColor: C.border, borderWidth: 1,
    borderRadius: 18, paddingHorizontal: 13, paddingVertical: 8,
  },
  activeChip: { backgroundColor: C.green, borderColor: C.green },
  chipText: { color: C.muted, fontSize: 12, fontWeight: '600' },
  activeChipText: { color: C.bg },
  listHeading: {
    paddingHorizontal: 20, marginTop: 15, marginBottom: 8,
    flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center',
  },
  sectionTitle: { color: C.text, fontSize: 19, fontWeight: '800' },
  muted: { color: C.muted, fontSize: 12, marginTop: 4 },
  sortRow: { flexDirection: 'row', gap: 7, paddingHorizontal: 16, marginBottom: 10 },
  sortChip: { padding: 8, borderRadius: 8, backgroundColor: C.card },
  sortActive: { borderColor: C.green, borderWidth: 1 },
  sortText: { color: C.muted, fontSize: 11 },
  list: { paddingHorizontal: 16, paddingBottom: 10 },
  gameCard: {
    flexDirection: 'row', alignItems: 'center', backgroundColor: C.card,
    borderWidth: 1, borderColor: C.border, borderRadius: 14,
    padding: 13, marginBottom: 10,
  },
  gameIcon: {
    width: 58, height: 58, borderRadius: 12,
    backgroundColor: '#263443', alignItems: 'center', justifyContent: 'center',
  },
  gameEmoji: { fontSize: 30 },
  gameInfo: { flex: 1, marginLeft: 13 },
  gameTitle: { color: C.text, fontSize: 16, fontWeight: '800', marginBottom: 4 },
  rating: { color: C.green, fontWeight: '800', marginTop: 6 },
  arrow: { color: C.green, fontSize: 28, paddingLeft: 8 },
  bottom: { borderTopWidth: 1, borderTopColor: C.border, padding: 12, alignItems: 'center' },
  bottomLink: { color: C.green, fontSize: 13, fontWeight: '700' },
  bottomCaption: { color: C.muted, fontSize: 10, marginTop: 6 },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 25 },
  error: { color: C.red, marginBottom: 12, textAlign: 'center' },
  empty: { color: C.muted, textAlign: 'center', marginTop: 35, padding: 20 },
  button: {
    backgroundColor: C.green, padding: 14, borderRadius: 11,
    alignItems: 'center', marginTop: 10,
  },
  secondaryButton: { backgroundColor: C.card, borderWidth: 1, borderColor: C.border },
  dangerButton: { backgroundColor: '#a93232' },
  buttonText: { color: C.bg, fontWeight: '800', fontSize: 14 },
  formContent: { padding: 18, paddingBottom: 35 },
  pageTitle: { color: C.text, fontSize: 26, fontWeight: '900', marginTop: 20 },
  field: { marginTop: 18 },
  label: { color: C.muted, fontSize: 12, fontWeight: '600', marginBottom: 8 },
  input: {
    backgroundColor: C.card, color: C.text, borderWidth: 1,
    borderColor: C.border, borderRadius: 11, padding: 13, fontSize: 15,
  },
  multiline: { minHeight: 90, textAlignVertical: 'top' },
  genreOptions: { flexDirection: 'row', flexWrap: 'wrap', gap: 8, marginTop: 4 },
  detailHero: {
    backgroundColor: C.card, borderRadius: 18, padding: 22,
    alignItems: 'center', marginVertical: 18,
  },
  detailTitle: {
    color: C.text, fontSize: 25, fontWeight: '900',
    textAlign: 'center', marginTop: 12,
  },
  detailRow: {
    backgroundColor: C.card, borderRadius: 10, padding: 14, marginTop: 8,
  },
  detailValue: { color: C.text, fontSize: 15, marginTop: 4 },
});
