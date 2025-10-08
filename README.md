<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Farcaster User Finders (<1k Obserwujących)</title>
    
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css" />

    <style>
        body {
            padding-bottom: 5rem;
        }
        .user-card {
            display: flex;
            align-items: center;
            gap: 1.5rem; /* Odstęp między elementami */
            margin-bottom: 1.5rem;
            padding: 1rem;
            border: 1px solid var(--pico-muted-border-color);
            border-radius: var(--pico-border-radius);
        }
        .user-card img {
            width: 70px;
            height: 70px;
            border-radius: 50%; /* Okrągły awatar */
        }
        .user-card .info {
            display: flex;
            flex-direction: column;
        }
        .user-card .info h3 {
            margin-bottom: 0.2rem;
        }
        .user-card .info a {
            text-decoration: none;
        }
        .user-card .followers {
            margin-left: auto; /* Przesuwa licznik na prawą stronę */
            font-weight: bold;
            font-size: 1.1em;
        }
        #loading {
            text-align: center;
            margin: 2rem 0;
        }
    </style>
</head>
<body>
    <main class="container">
        <header style="text-align: center; margin: 2rem 0;">
            <h1>🚀 Farcaster User Finder</h1>
            <p>Znajdź aktywnych użytkowników z mniej niż 1000 obserwujących.</p>
        </header>

        <article>
            <div style="text-align: center;">
                <button id="searchButton">Szukaj użytkowników</button>
            </div>
            
            <div id="loading" style="display: none;">
                <progress></progress>
                <p>Przeszukuję sieć Farcaster...</p>
            </div>

            <div id="results" style="margin-top: 2rem;"></div>
        </article>
    </main>

    <script>
        // --- KONFIGURACJA ---
        const NEYNAR_API_KEY = "TUTAJ_WKLEJ_SWOJ_KLUCZ_API_NEYNAR";
        const MAX_FOLLOWERS = 1000;
        const USERS_TO_FETCH = 100; // Ile profili pobrać do przefiltrowania

        // --- ELEMENTY STRONY (DOM) ---
        const searchButton = document.getElementById('searchButton');
        const loadingDiv = document.getElementById('loading');
        const resultsDiv = document.getElementById('results');

        // --- LOGIKA APLIKACJI ---

        // Nasłuchuj kliknięcia na przycisk
        searchButton.addEventListener('click', findUsers);

        /**
         * Główna funkcja do wyszukiwania i filtrowania użytkowników
         */
        async function findUsers() {
            if (!NEYNAR_API_KEY || NEYNAR_API_KEY === "TUTAJ_WKLEJ_SWOJ_KLUCZ_API_NEYNAR") {
                resultsDiv.innerHTML = `<p style="color: red; text-align: center;"><strong>Błąd:</strong> Nie podano klucza API Neynar w kodzie!</p>`;
                return;
            }

            // Pokaż animację ładowania i wyczyść stare wyniki
            loadingDiv.style.display = 'block';
            resultsDiv.innerHTML = '';
            searchButton.setAttribute('aria-busy', 'true');
            searchButton.disabled = true;

            const foundUsers = new Map(); // Używamy Map, aby uniknąć duplikatów
            
            try {
                // Budujemy URL do API Neynar
                const apiUrl = `https://api.neynar.com/v2/farcaster/feed/popular?limit=${USERS_TO_FETCH}`;
                
                const response = await fetch(apiUrl, {
                    method: 'GET',
                    headers: {
                        'accept': 'application/json',
                        'api_key': NEYNAR_API_KEY
                    }
                });

                if (!response.ok) {
                    throw new Error(`Błąd API: ${response.statusText}`);
                }

                const data = await response.json();
                const casts = data.casts || [];

                for (const cast of casts) {
                    const author = cast.author;

                    // Sprawdzamy, czy użytkownik ma mniej niż 1000 followersów i czy nie ma go już na liście
                    if (author.follower_count < MAX_FOLLOWERS && !foundUsers.has(author.fid)) {
                        foundUsers.set(author.fid, author);
                    }
                }
                
                displayUsers(Array.from(foundUsers.values()));

            } catch (error) {
                console.error("Wystąpił błąd:", error);
                resultsDiv.innerHTML = `<p style="color: red; text-align: center;">Nie udało się pobrać danych. Spróbuj ponownie. Błąd: ${error.message}</p>`;
            } finally {
                // Ukryj ładowanie i odblokuj przycisk
                loadingDiv.style.display = 'none';
                searchButton.setAttribute('aria-busy', 'false');
                searchButton.disabled = false;
            }
        }

        /**
         * Funkcja do wyświetlania znalezionych użytkowników na stronie
         * @param {Array} users - Tablica obiektów użytkowników
         */
        function displayUsers(users) {
            if (users.length === 0) {
                resultsDiv.innerHTML = '<p style="text-align: center;">Nie znaleziono żadnych użytkowników spełniających kryteria. Spróbuj ponownie!</p>';
                return;
            }
            
            // Sortujemy użytkowników od najmniejszej liczby obserwujących
            users.sort((a, b) => a.follower_count - b.follower_count);

            let html = '';
            for (const user of users) {
                html += `
                    <div class="user-card">
                        <img src="${user.pfp_url}" alt="Awatar ${user.display_name}">
                        <div class="info">
                            <a href="https://warpcast.com/${user.username}" target="_blank">
                                <h3>${user.display_name}</h3>
                            </a>
                            <p>@${user.username}</p>
                        </div>
                        <div class="followers">
                            ${user.follower_count} followers
                        </div>
                    </div>
                `;
            }
            resultsDiv.innerHTML = html;
        }
    </script>
</body>
</html>
