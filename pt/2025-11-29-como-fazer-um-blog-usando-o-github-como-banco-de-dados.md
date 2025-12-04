# Como fazer um blog usando o Github como banco de dados

### Porque eu queria tentar algo diferente

Eu já tinha visto essa ideia antes, infelizmente não lembro onde. Provavelmente no blog de alguém. No final de cada post, havia um link para o arquivo markdown daquele post no Github, e qualquer pessoa podia oferecer um pull request para aquele arquivo para sugerir uma correção. Eu gostei da ideia, mas pensei "por que não ir até o fim e usar o Github como o armazenamento dos posts?". Eu também queria um projeto pessoal para usar Laravel. Eu já uso Laravel profissionalmente para APIs, mas queria tentar um sistema simples usando suas principais funcionalidades, com foco no Blade e markdown.

Então a ideia era simples: sem banco de dados, uma lista simples de posts, e o conteúdo de cada post armazenado em um arquivo markdown no Github. Usar a API do Github para buscar tudo, e renderizar o markdown de cada post dinamicamente usando Blade.

### Vamos ver o código

```php
class PostController extends Controller
{

    public function __construct(
        private PostService $postService,
    ) {}

    public function index()
    {
        $posts = $this->postService->getPosts();

        return view('home', [
            'posts' => $posts,
        ]);
    }

    public function show(string $locale, string $slug)
    {
        $post = $this->postService->getPostBySlug($slug);
        $markdown = base64_decode($post->content);
        $html = Markdown::parse($markdown);
        return view('post', compact('html'));
    }
}
```

Nada de muito especial aqui, a única coisa de destaque é o uso de `Markdown::parse($markdown)` para 'transformar' o markdown em HTML antes de enviá-lo para a view.

```php
class PostService
{
    public function __construct(
        private GithubService $githubService,
    ) {}

    public function getPosts(): Collection
    {
        $posts = $this->githubService->getPosts()
            ->map(function ($item) {
                return [
                    'title' => $this->getMarkdownTitle($item['download_url']),
                    'subtitle' => $this->getMarkdownSubtitle($item['download_url']),
                    'date' => $this->getPostDate($item['name']),
                    'name' => $this->getPostName($item['name']),
                    'slug' => $item['name'],
                    'download_url' => $item['download_url'],
                ];
            })
            ->sortByDesc('date');

        return $posts;
    }

    public function getPostBySlug(string $slug)
    {
        $post = $this->githubService->getPostBySlug($slug);
        return $post;
    }
}
```

Uma leve manipulação dos dados é feita na classe `PostService`, para extrair o título, subtítulo, data, nome e slug da resposta da API do Github.

```php
class GithubService
{
    // Constructor, configuration, etc.

    public function getPosts(): Collection
    {
        $posts = Http::withHeaders([
            'Authorization' => "token {$this->token}",
            'Accept' => 'application/vnd.github.v3+json',
        ])->get("https://api.github.com/repos/{$this->username}/{$this->repo}/contents/{$this->locale}")
            ->throw()
            ->collect();

        return $posts;
    }

    public function getPostBySlug(string $slug)
    {
        $post = Http::withHeaders([
            'Authorization' => "token {$this->token}",
            'Accept' => 'application/vnd.github.v3+json',
        ])->get("https://api.github.com/repos/{$this->username}/{$this->repo}/contents/{$this->locale}/{$slug}")
            ->throw()
            ->object();

        return $post;
    }
}
```

Finalmente, `GithubService` é apenas um wrapper para a API do Github, usando a facade `Http` do Laravel para fazer as requisições.

Mas como o post aparece na view?

```php
<x-layout>
    <div class="max-w-3xl lg:max-w-4xl flex flex-col gap-6">
        <article class="prose dark:prose-invert max-w-none">
            {!! $html !!}
        </article>
    </div>
</x-layout>
```

Não poderia ser mais simples! O ingrediente secreto aqui é usar a classe `prose` do Tailwind CSS, que fornece uma estilização completa e pronta para uso para o html simples em que o markdown é convertido.

Espero que isso te inspire a tentar algo parecido! Acho que tenho um blog agora, então espere mais posts como este explorando alguns projetos simples e divertidos que estou construindo. Obrigado pela leitura!
