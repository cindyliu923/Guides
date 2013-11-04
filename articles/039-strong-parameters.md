# [Strong Parameters](https://github.com/rails/strong_parameters)

Rails 4 引進了一個新的保護機制：Strong Parameters，譯作健壯參數、強壯參數等。

但我認為譯作“合法參數”、“核可參數”較好。

## 什麼是 Strong Parameters?

在建模或更新 Active Record 物件時，會有 Mass assignment 的問題。

Mass assignment 就是建模或更新時，傳入 hash 參數，一次給多個欄位賦值：

```ruby
# params[:book] => { name: 'Ruby on Rails 4', who: 'Juanito Fatas', role: :reviewer }

def create
  @ruby_on_rails4 = Book.create(params[:book])
end

def update
  @ruby_on_rails4 = Book.update(params[:book])
end
```

這裡看到角色是 Reviewer，若角色被惡意改成擁有最高權限的角色，如作者，則會有安全性問題。

此時便需要建立白名單機制 ＋ `attr_accessible` 或是 `attr_protected` 來確保允許哪些參數可以大量賦值。

更詳細的內容可以參考業界先進[張文鈿（_ihower_）先生](https://ihower.tw/)於 [《Ruby on Rails 實戰聖經》網路安全](http://ihower.tw/rails3/security.html)一章，關於大量賦值（Mass assignment）的說明。

Rails 4 推出了 Strong Parameters。

## Strong Parameters 例子

現在所有參數的核定，交由 Controller 處理，由 Controller 決定，哪些參數可以大量賦值：

```ruby
class BookController < ActionController::Base
  def create
    Book.create(person_params)
  end

  def update
    book = Book.find(params[:id])
    book.update(book_params)
    redirect_to book
  end

  private

    def book_params
      params.require(:book).permit(:name, :who)
    end
end
```

原本 `create`、`update` 方法直接傳入整個 `params`，現在傳入一個封裝過的 `book_params`

`book_params` 是個 private 方法，在這寫，需要（`require`）用到那個 model，並允許（`permit`）哪些欄位做大量賦值。

若試圖傳入未允許的欄位，會拋 `ActiveModel::ForbiddenAttributesError` 錯誤。

### Strong Parameters 語法：

__params.需要(:model_name).允許(:欄位_1, :欄位_2)__

```ruby
params.require(:person).permit(:name, :age)
```

基本上就是這樣用，依此類推。

## `params`

平常 Controller 可取用的 `params` 是 [ActionController::Parameters](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html) 的 instance：

```ruby
params = ActionController::Parameters.new(name: 'Juanito Fatas')
```

## [Strong Parameters API](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html) 概觀

## [require(key)](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html#method-i-require)

接受一個參數：`key`。先檢查 params 是否存在，存在時返回符合 `key` 的 hash；否則拋出 `ActionController::ParameterMissing` 錯誤。

```ruby
# rails console
> params = ActionController::Parameters.new(bridegroom: { name: 'Juanito Fatas', age: 42 }, bride: { name: '蒼井そら', age: 18 })

> params.require(:bridegroom) # => { name: 'Juanito Fatas', age: 42 }

> params.require(:bride)      # => { name: '蒼井そら', age: 18 }
```

## [permit](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html#method-i-permit)

返回 1 個__新的 ActionController::Parameters__ instance，僅帶有允許的屬性。

```ruby
# rails console
> params = ActionController::Parameters.new(bridegroom: { name: 'Juanito Fatas', age: 42 }, bride: { name: '蒼井そら', age: 18 })

> params.require(:bride).permit(:name)
Unpermitted parameters: age
=> { 'name' => '蒼井そら' }

> params.require(:bride).permit(:name, :age).is_a? ActionController::Parameters
=> true

> params.require(:bride).is_a? ActionController::Parameters
=> true
```

由於是 ActionController::Parameters 的 instance，所以 `permit` 可以連鎖使用。

傳入沒有的參數也沒關係：

```ruby
> params.require(:bride).permit(:name, :age, :cup)
=> { 'name' => '蒼井そら', "age" => 18 }
```

但若真需要知道某個參數，用 `require`：

```ruby
> params.require(:bride).permit(:name, :age).require(:cup)
ActionController::ParameterMissing: param not found: cup
```

`require` 返回 hash 對應 key 的值，`permit` 返回 ActionController::Parameters 的 instance。

```ruby
>  params.require(:bride).require(:name)
=> '蒼井そら'
>  params.require(:bride).permit(:name)
=> { 'name' => '蒼井そら' }
```

## [permitted?](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html#method-i-permitted-3F)

檢查 `params` 是否允許大量賦值，允許返回真，否則假。

```ruby
> params.require(:bride).permitted?
=> false
> params.require(:bride).permit(:name).permitted?
=> true
```

看另外一個例子，我跟蒼井小姐是否可結婚？

```ruby
> params = ActionController::Parameters.new(newlywed_couple: ['Juanito Fatas', '蒼井そら'])
=> { "newlywed_couple" => ["Juanito Fatas", "蒼井そら"] }
> params.permit(:newlywed_couple)
=> Unpermitted parameters: newlywed_couple
```

原來少了棟房子：

```
> params.permit(newlywed_couple: [])
=> { "newlywed_couple" => ["Juanito Fatas", "蒼井そら"] }
```

__要告訴 Strong Parameters key 是 Array。__

那要是參數是 nested hash? 平常 Hash 怎麼取值，便怎麼取：

```ruby
> params = ActionController::Parameters.new(newlywed_couple: { bridegroom: 'Juanito Fatas', bride: '蒼井そら' })
=> { "newlywed_couple" => { "bridegroom" => "Juanito Fatas", "bride"=>"蒼井そら"} }
> params.permit(newlywed_couple: [:bride])
=> { "newlywed_couple" => { "bride" => "蒼井そら"} }
```

用 `require` ＋ `permit`:

```ruby
> params.require(:newlywed_couple).permit(:bride)
Unpermitted parameters: bridegroom
=> { "bride" => "蒼井そら" }
```

更複雜的例子：

```ruby
> params = ActionController::Parameters.new(user: { username: "john", data: { foo: "bar" } })
=> {"user"=>{"username"=>"john", "data"=>{"foo"=>"bar"}}}
```

假設我們不知道 `data` hash 裡有什麼 key，這該怎麼辦？

```ruby
> params.require(:user).permit(:username).tap do |whitelisted|
    whitelisted[:data] = params[:user][:data]
  end
Unpermitted parameters: data
=> { "username" => "john", "data" => {"foo"=>"bar"}}
```

要是知道有什麼 key:

```ruby
params.require(:user).permit(:username, data: [ :foo ])
```

### [permit!](http://edgeapi.rubyonrails.org/classes/ActionController/Parameters.html#method-i-permit-21)

這個方法很危險，我不告訴你怎麼用。

## 延伸閱讀

[ActionController::StrongParameters](http://edgeapi.rubyonrails.org/classes/ActionController/StrongParameters.html)

[Rails 4: Testing strong parameters - Pivotal Labs](http://pivotallabs.com/rails-4-testing-strong-parameters/)