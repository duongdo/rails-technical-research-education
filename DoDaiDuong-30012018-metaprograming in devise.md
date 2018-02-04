# Metaprograming

Metaprograming trong Devise
Metaprogramming hiểu đơn giản là "Code sinh ra code" nghĩa là mình viết một chương trình và chương trình này sẽ sinh ra, điều khiển các chương trình khác hoặc làm 1 phần công việc ở thời điểm biên dịch
Metaprogramming là kỹ thuật viết code để quản lý code
Devise sử dụng metaproraming để tạo ra các code và method dựa vào ứng dụng của ta. Khi ta tạo controller của Devise, sẽ có rất nhiều code dùng để tìm tên model và tạo các biến instance dù không biết tên của chúng. Đây cũng là lý do bạn có thể dùng @users hoặc bất kỳ tên nào dựa vào model bạn khai báo mà vẫn chạy bình thường.


## Một số đoạn Devise Metaprograming:

Khai báo routes của Devise `\lib\devise\controllers\url_helpers.rb`
+ Ở đây Devise tạo vòng lặp và define_method để tạo ra các method cơ bản chung nhất cho các routes

```ruby
def self.generate_helpers!(routes=nil)
  routes ||= begin
    mappings = Devise.mappings.values.map(&:used_helpers).flatten.uniq
    Devise::URL_HELPERS.slice(*mappings)
  end

  routes.each do |module_name, actions|
    [:path, :url].each do |path_or_url|
      actions.each do |action|
        action = action ? "#{action}_" : ""
        method = :"#{action}#{module_name}_#{path_or_url}"

        define_method method do |resource_or_scope, *args|
          scope = Devise::Mapping.find_scope!(resource_or_scope)
          router_name = Devise.mappings[scope].router_name
          context = router_name ? send(router_name) : _devise_route_context
          context.send("#{action}#{scope}_#{module_name}_#{path_or_url}", *args)
        end
      end
    end
  end
end
```

Tạo các method cho 1 nhóm khi cần mapping nhiều đối tượng một lúc
+ Ở đây sử dụng class_eval (để tạo các phương thức class) có thể gồm một khối (trong đó có thiết lập self của đối tương mà bạn đang sử dụng), hoặc một chuỗi các mã
`\lib\devise\controllers\helpers.rb`
```ruby
def devise_group(group_name, opts={})
  mappings = "[#{ opts[:contains].map { |m| ":#{m}" }.join(',') }]"

  class_eval <<-METHODS, __FILE__, __LINE__ + 1
    def authenticate_#{group_name}!(favourite=nil, opts={})
      unless #{group_name}_signed_in?
        mappings = #{mappings}
        mappings.unshift mappings.delete(favourite.to_sym) if favourite
        mappings.each do |mapping|
          opts[:scope] = mapping
          warden.authenticate!(opts) if !devise_controller? || opts.delete(:force)
        end
      end
    end

    def #{group_name}_signed_in?
      #{mappings}.any? do |mapping|
        warden.authenticate?(scope: mapping)
      end
    end

    def current_#{group_name}(favourite=nil)
      mappings = #{mappings}
      mappings.unshift mappings.delete(favourite.to_sym) if favourite
      mappings.each do |mapping|
        current = warden.authenticate(scope: mapping)
        return current if current
      end
      nil
    end

    def current_#{group_name.to_s.pluralize}
      #{mappings}.map do |mapping|
        warden.authenticate(scope: mapping)
      end.compact
    end

    if respond_to?(:helper_method)
      helper_method "current_#{group_name}", "current_#{group_name.to_s.pluralize}", "#{group_name}_signed_in?"
    end
  METHODS
end
end
```

Tạo các phương thức cho từng role như:

  Ta có 2 role : user, admin thì ta sẽ có một số phương thức như sau
+ user_signed_in?, admin_signed_in?
+ current_user, current_admin
+ user_session, admin_session
+ authenticate_user!, authenticate_admin!
```ruby
def self.define_helpers(mapping) #:nodoc:
  mapping = mapping.name

  class_eval <<-METHODS, __FILE__, __LINE__ + 1
    def authenticate_#{mapping}!(opts={})
      opts[:scope] = :#{mapping}
      warden.authenticate!(opts) if !devise_controller? || opts.delete(:force)
    end

    def #{mapping}_signed_in?
      !!current_#{mapping}
    end

    def current_#{mapping}
      @current_#{mapping} ||= warden.authenticate(scope: :#{mapping})
    end

    def #{mapping}_session
      current_#{mapping} && warden.session(:#{mapping})
    end
  METHODS

  ActiveSupport.on_load(:action_controller) do
    if respond_to?(:helper_method)
      helper_method "current_#{mapping}", "#{mapping}_signed_in?", "#{mapping}_session"
    end
  end
end
```
