# Strategies in Warden

Warden sử dụng khái niệm Strategies để xác định nếu có request authenticated. Warden sẽ trả về 1 trong 3 trạng thái:
One succeeds
No Strategies are found relevant
A strategy Fails
* Warden dùng strategies để authenticate nhưng nó lại không hề tạo ra cho bạn bất kì một strategies nào mà thay vào đó bạn phải tự tạo và tự code.
Strategies là nơi để bạn đặt code logic cho một authenciate request.
Nó là lớp kế thừa của Warden::Strategies::Base
Đây là 1 ví dụ về Strategies có tên là password
```ruby
Warden::Strategies.add(:password) do

  def valid?
    params['username'] || params['password']
  end

  def authenticate!
    u = User.authenticate(params['username'], params['password'])
    u.nil? ? fail!("Could not log in") : success!(u)
  end
end
```
Ở đây ta cần chú ý đến 2 method `valid?` và `authenticate!`
`valid?`
Method valid? dùng để bảo vệ các Strategies. Nếu ta không khai báo valid? thì Strategy sẽ luôn chạy. Còn nếu ta khai báo valid? thì Strategy sẽ chỉ chạy khi valid? trả về true
Theo ví dụ ở trên thì nếu có username hoặc password mà user đang cố gắng đăng nhập. Khi chỉ có username hoặc password thì việc gọi User.authenticate sẽ thất bại, nhưng nó vẫn được tính hợp lệ do valid?
`authenticate!`
Đây là nơi xác thực các request. Đây là nơi thiết lập các logic để xác thực các request sẽ xảy ra.

Các request hợp lệ sẽ gồm:
+ Request: Rack::Request object
+ Env:  Rack env object
+ Session: session object của request
+ Params: params của request

Các action sẽ được thực hiện trong Strategy
+ halt!: dừng Strategy. Cho Strategy xử lý lần cuối cùng
+ pass: bỏ qua Strategy. Cái này không bắt buộc phải khai báo
+ success!: dùng khi có request đăng nhập với user object . Một trong các nguyên nhân gây ra halt!
+ fail!: thiết lập Strategy fail. Một trong các nguyên nhân gây ra halt!. Ta sẽ phải tự throw :warden symbol để buộc action này chạy
+ redirect!: chuyển hướng sang url khác. Ta có thể truyền params vào để mã hóa và lựa chọn
+ custom!: trả về một mảng rack tùy chỉnh để handle lại. Một trong các nguyên nhân gây ra halt!

Bên cạnh đó còn một số action như:
+ header: thiết lập header để phản hồi lại Strategy
+ errors: cung cấp truy cập đến errors object. Ở đây bạn có thể cung cấp các errors liên quan đến authenticate


Bây giờ ta sẽ xem Devise tạo một Strategy của Warden như thế nào
Ta xem lib\devise\strategies của Devise:
## `base.rb`
```ruby
module Devise
  module Strategies
    # Base strategy for Devise. Responsible for verifying correct scope and mapping.
    class Base < ::Warden::Strategies::Base
      # Whenever CSRF cannot be verified, we turn off any kind of storage
      def store?
        !env["devise.skip_storage"]
      end

      # Checks if a valid scope was given for devise and find mapping based on this scope.
      def mapping
        @mapping ||= begin
          mapping = Devise.mappings[scope]
          raise "Could not find mapping for #{scope}" unless mapping
          mapping
        end
      end
    end
  end
end
```
- Ta có thể thấy Class Base kế thừa từ Warden::Strategies::Base của Warden
- Base strategy sẽ đảm nhiệm vai trò xác minh đúng scope và mapping
+ store?: Khi không xác minh được CSRF(Cross Site Request Forgery), ta sẽ đóng tất cả có storage
+ mapping: tìm mapping dựa vào scope được truyền vào. Nếu mapping không tồn tại sẽ trả về "Could not find mapping for #{scope}"

## `authenticatable.rb`
`class Authenticatable < Base`
Authenticatable kế thừa từ lớp Base:
Đây là method valid? mà ta đã đề cập trước đó: ta có thể thấy nó sẽ gọi 2 method là valid_for_params_auth? và valid_for_http_auth?
```ruby
def valid?
  valid_for_params_auth? || valid_for_http_auth?
end
```
- valid_for_params_auth? sẽ kiểm tra http request có hợp lệ không bằng cách:

```ruby
def valid_for_params_auth?
  params_authenticatable? && valid_params_request? &&
    valid_params? && with_authentication_hash(:params_auth, params_auth_hash)
end
```
  + Xác nhận nếu model cho phép xác thực http
  + Nếu có bất kỳ header nào được gửi
  + Nếu tất cả authenticate key được xác thực
- valid_for_http_auth? sẽ kiểm tra params truyền lên có hợp lệ hay không bằng cách
```ruby
def valid_for_http_auth?
  http_authenticatable? && request.authorization && with_authentication_hash(:http_auth, http_auth_hash)
end
```
  + Xác nhận nếu model cho phép xác thực params
  + Nếu yêu cầu quyền truy cập session controller  thông qua POST
  + Nếu params[scope] trả về một hash với các thông tin
  + Nếu tất cả authenticate key được xác thực
-Chúng ta sẽ cần chú ý đến method with_authentication_hash vì cả 2 method valid đều gọi đến: method này sẽ thiết lập authenticate hash và password cho params_auth_hash và http_auth_hash
```ruby
def with_authentication_hash(auth_type, auth_values)
  self.authentication_hash, self.authentication_type = {}, auth_type
  self.password = auth_values[:password]

  parse_authentication_key_values(auth_values, authentication_keys) &&
  parse_authentication_key_values(request_values, request_keys)
end
```

## `database_authenticatable.rb`
`class DatabaseAuthenticatable < Authenticatable`
- DatabaseAuthenticatable sẽ kế thừa từ Authenticatable: vậy là DatabaseAuthenticatable sẽ nhận valid? của Authenticatable, bây giờ ta sẽ xem xem method authenticate! có được khai báo ở đây không nha

```ruby
def authenticate!
  resource  = password.present? && mapping.to.find_for_database_authentication(authentication_hash)
  hashed = false

  if validate(resource){ hashed = true; resource.valid_password?(password) }
    remember_me(resource)
    resource.after_database_authentication
    success!(resource)
  end

  mapping.to.new.password = password if !hashed && Devise.paranoid
  fail(:not_found_in_database) unless resource
end
```
  + Ở đây sẽ kiểm tra password và authentication hash được tạo bởi with_authentication_hash có tồn tại hay không
  + Nếu validate(resource) trả về true thì gọi success! của Warden để cung cấp 1 user object từ resource và cho Strategy xử lý lần cuối cùng
Cuối cùng Devise sẽ thêm vào Strategy của Warden
```ruby
Warden::Strategies.add(:database_authenticatable, Devise::Strategies::DatabaseAuthenticatable)
```
Vậy đây là một Strategy mà Devise tạo ra ở Warden

