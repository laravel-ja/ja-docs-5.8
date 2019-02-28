# コンソールテスト

- [イントロダクション](#introduction)
- [入出力の期待値](#expecting-input-and-output)

## イントロダクション

HTTPテストを簡単にするのに加えて、ユーザー入力を尋ねるコンソールアプリケーションテストに対するシンプルなAPIも、Laravelは提供しています。

<a name="expecting-input-and-output"></a>
## 入出力の期待値

Laravelで`expectsQuestion`メソッドを使用すれば、コンソールコマンドのユーザー入力を簡単に「モック」できます。更に、終了コードを`assertExitCode`メソッドで、コンソールコマンドに期待する出力を`expectsOutput`メソッドで指定することもできます。

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you program in?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you program in '.$language.'.');
    });

このコマンドを以下のように、`expectsQuestion`、`expectsOutput`、`assertExitCode`メソッドを活用してテストできます。

    /**
     * コンソールコマンドのテスト
     *
     * @return void
     */
    public function test_console_command()
    {
        $this->artisan('question')
             ->expectsQuestion('What is your name?', 'Taylor Otwell')
             ->expectsQuestion('Which language do you program in?', 'PHP')
             ->expectsOutput('Your name is Taylor Otwell and you program in PHP.')
             ->assertExitCode(0);
    }


