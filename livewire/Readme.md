# LiveWire コンポーネント

## Livewire setup
Livewire を設定するときに最初に行うことは、スタイルとスクリプトを追加することです。

Breeze をインストールすると、それらを追加するための便利なレイアウト ファイルが作成されます。
そのファイルは次のとおりです　`resources/views/layouts/app.blade.php`

```bash
vim resources/views/layouts/app.blade.php
```
`</head>` 直前に `@livewireStyles` を追記
`</body>` 直前に `@livewireScripts` `@stack('scripts')` を追記
```php
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="csrf-token" content="{{ csrf_token() }}">

        <title>{{ config('app.name', 'Laravel') }}</title>

        <!-- Fonts -->
        <link rel="preconnect" href="https://fonts.bunny.net">
        <link href="https://fonts.bunny.net/css?family=figtree:400,500,600&display=swap" rel="stylesheet" />

        <!-- Scripts -->
        @vite(['resources/css/app.css', 'resources/js/app.js'])
        <!-- Livewire -->
        @livewireStyles
    </head>
    <body class="font-sans antialiased">
        <div class="min-h-screen bg-gray-100">
            @include('layouts.navigation')

            <!-- Page Heading -->
            @if (isset($header))
                <header class="bg-white shadow">
                    <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
                        {{ $header }}
                    </div>
                </header>
            @endif

            <!-- Page Content -->
            <main>
                {{ $slot }}
            </main>
        </div>
        <!-- Livewire -->
        @livewireScripts
        <!-- Scripts -->
        @stack('scripts')
    </body>
</html>
```

## Livewire Component
Livewire コンポーネントを作成しましょう。
このコンポーネントは単なるボタンになります。
このボタンは、グローバル通知に通知を表示するように指示するイベントを送出します。

次のコマンドを実行してコンポーネントを作成します。
```bash
php artisan livewire:make ClickyButton
```
これにより、次のファイルが生成されます。
- `resources/views/livewire/clicky-button.blade.php`
- `app/Http/Livewire/ClickyButton.php`

blade view にボタンを表示するコードを記述します。
```bash
vim resources/views/livewire/clicky-button.blade.php
```
```php
<div>
    <button
        wire:click="tellme"
        class="rounded border px-4 py-2 bg-indigo-500 text-white"
    >
        Tell me something
    </button>
</div>

@push('scripts')
<script>
    window.addEventListener('notify', event => {
        alert(event.detail);
    });
</script>
@endpush
```

ボタンをクリックした時に処理する function 記述します。
```bash
vim app/Http/Livewire/ClickyButton.php
```
```php
<?php

namespace App\Http\Livewire;

use Livewire\Component;

class ClickyButton extends Component
{
    public function tellme()
    {
        $messages = [
            'A blessing in disguise',
            'Bite the bullet',
            'Call it a day',
            'Easy does it',
            'Make a long story short',
            'Miss the boat',
            'To get bent out of shape',
            'Birds of a feather flock together',
            "Don't cry over spilt milk",
            'Good things come',
            'Live and learn',
            'Once in a blue moon',
            'Spill the beans',
        ];


        $this->dispatchBrowserEvent(
            'notify', 
            $messages[array_rand($messages)]
        );
    }
    public function render()
    {
        return view('livewire.clicky-button');
    }
}
```

`ClickyButton` コンポーネントをクリックできるようにアプリケーションにコンポーネントを追加します。

ユーザーがログイン後に表示されるダッシュボードに追加します。
```bash
vim resources/views/dashboard.blade.php
```
`@livewire('clicky-button')` を記述します。
```php
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">
            {{ __('Dashboard') }}
        </h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
                <div class="p-6 text-gray-900">
                    {{ __("You're logged in!") }}
                    <br><br>
                    @livewire('clicky-button')
                </div>
            </div>
        </div>
    </div>
</x-app-layout>
```
ここまでの作業で、ブラウザーで表示と、クリック時に、alert 表示が出来る様になります。

## 通知表示用コンポーネント作成
前項で作成したイベントをリッスンして、通知を表示する必要があります。

これを実現するために、コンポーネントを作成します。
`Livewire` コンポーネントではありません。
代わりに、`Laravel` コンポーネントであり、基本的には単なる別のビュー ファイルです。

次のコマンドを実行してコンポーネントを作成します。
```bash
php artisan make:component Notifications
```
これにより、次のファイルが生成されます。
- `resources/views/components/notifications.blade.php`
- `app/View/Components/Notifications.php`

コンポーネントの view を記述して行きます。
```bash
vim resources/views/components/notifications.blade.php
```
```php
<div
    x-data="{
        messages: [],
        remove(message) {
            this.messages.splice(this.messages.indexOf(message), 1)
        },
    }"
    @notify.window="let message = $event.detail; messages.push(message); setTimeout(() => { remove(message) }, 2500)"
    class="z-50 fixed inset-0 flex flex-col items-end justify-center px-4 py-6 pointer-events-none sm:p-6 sm:justify-start space-y-4"
>
    <template x-for="(message, messageIndex) in messages" :key="messageIndex" hidden>
        <div
            class="max-w-sm w-full bg-white shadow-lg rounded-lg pointer-events-auto"
        >
            <div class="rounded-lg shadow-lg overflow-hidden">
                <div class="p-4">
                    <div class="flex items-start">
                        <div class="flex-shrink-0">
                            <svg class="h-6 w-6 text-green-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"></path>
                            </svg>
                        </div>
                        <div class="ml-3 w-0 flex-1 pt-0.5">
                            <p x-text="message" class="text-sm leading-5 font-medium text-gray-900"></p>
                        </div>
                        <div class="ml-4 flex-shrink-0 flex">
                            <button @click="remove(message)" class="inline-flex text-gray-400 focus:outline-none focus:text-gray-500 transition ease-in-out duration-150">
                                <svg class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                                    <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"></path>
                                </svg>
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </template>
</div>
```

通知表示用コンポーネントを認証後の全頁で組み込みたいので以下のレイアウトファイルに
コンポーネントを読み込みコードを記述します。
 `resources/views/layouts/app.blade.php`
 ```bash
 vim resources/views/layouts/app.blade.php
 ```
 `<x-notifications />`を追記します。
 ```php
 <!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="csrf-token" content="{{ csrf_token() }}">

        <title>{{ config('app.name', 'Laravel') }}</title>

        <!-- Fonts -->
        <link rel="preconnect" href="https://fonts.bunny.net">
        <link href="https://fonts.bunny.net/css?family=figtree:400,500,600&display=swap" rel="stylesheet" />

        <!-- Scripts -->
        @vite(['resources/css/app.css', 'resources/js/app.js'])
        <!-- Livewire -->
        @livewireStyles
    </head>
    <body class="font-sans antialiased">
        <div class="min-h-screen bg-gray-100">
            @include('layouts.navigation')

            <!-- Page Heading -->
            @if (isset($header))
                <header class="bg-white shadow">
                    <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
                        {{ $header }}
                    </div>
                </header>
            @endif

            <!-- Page Content -->
            <main>
                <x-notifications />
                {{ $slot }}
            </main>
        </div>
        <!-- Livewire -->
        @livewireScripts
        <!-- Scripts -->
        @stack('scripts')
    </body>
</html>
 ```

修正 `@push('script')` の記述を削除します。
```bash
vim resources/views/livewire/clicky-button.blade.php
```
`script` 内の イベントリスナー処理は、コンポーネントの Alipne.js にて処理します。
```php
<div>
    <button
        wire:click="tellme"
        class="rounded border px-4 py-2 bg-indigo-500 text-white"
    >
        Tell me something
    </button>
</div>
@endpush
```
## AlpineJS
前項のコンポーネントについて
1. ブラウザー右上に通知を表示
2. AlpineJSを使用して、メッセージを表示して、2.5秒後に非表示にします。
3. AlpineJSを使用して、`<template>` タグに `messages` 配列をループ処理して、動的に表示します。

`x-data` にて DOM要素にプロパティとメソッドを記述します。

イベントリスナー処理は、`@notify.window` はイベントリスナになります。

`<template>` にてDOM要素を動的にテンプレート化して表示処理をします。

## 参考サイト
- https://fly.io/laravel-bytes/global-notifications-with-livewire/
- https://readouble.com/livewire/2.x/ja/rendering-components.html
- https://codepen.io/KevinBatdorf/pen/JjGKbMa?editors=1010