# How to make a blog using Github as a database

### Cause I wanted to try something different

I've seen this idea done before, unfortunately I don't remember where. Someone's blog probably. At the end of each post, there was a link to the markdown file of that post on Github, and anyone could offer a pull request to that file to suggest a correction. I liked the idea, but thought "why not just go all the way and use Github as the post storage?". I also wanted a pet project to use Laravel. I already use Laravel a lot for APIs, but wanted to try a simple system using its main features, with a focus on Blade and markdown.

So the idea was simple: no database, a simple list of blog posts, and the content of each post stored in a markdown file on Github. Use Github API to fetch everything, and render each post's markdown on the fly using Blade.

### Let's see the code

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

Nothing too fancy here, the only thing that stands out is the use of `Markdown::parse($markdown)` to 'transform' the markdown to HTML before sending it to the view.

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

Some manipulation of the data is done in the `PostService` class, to extract the title, subtitle, date, name and slug from the Github API response.

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

Finally, `GithubService` is just a wrapper for the Github API, using Laravel's `Http` facade to make the requests.

But how does the post show up in the view?

```php
<x-layout>
    <div class="max-w-3xl lg:max-w-4xl flex flex-col gap-6">
        <article class="prose dark:prose-invert max-w-none">
            {!! $html !!}
        </article>
    </div>
</x-layout>
```

Couldn't be simpler! The secret sauce here is using the `prose` class from Tailwind CSS, which provides an out-of-the-box complete stylization to the simple html the markdown converts to. 

I hope this inspires you to give something similar a try! I guess I have a blog now, so expect more posts like this exploring some simple and fun projects I'm building. Thanks for reading!


