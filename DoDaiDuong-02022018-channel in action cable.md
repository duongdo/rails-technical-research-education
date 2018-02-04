## Action Cable

Tích hợp Websocket cho rails. Nó kết nối Websocket với các ứng dụng của Rails sử dụng kiểu kết nối stateful  không giống như HTTP request. chi phí, độ trễ liên quan đến việc đẩy dữ liệu trong thời gian thực được giảm đáng kể so với HTTP.

Có 2 khái niệm ta cần biết là:
- Stateful là kiểu kết nối mà server cần lưu dữ liệu của client, như vậy ràng buộc giữa client và server vẫn được giữ sau mỗi request của client. Dữ liệu được server lưu lại có thể làm đầu vào (input parameters) cho lần  kế tiếp, hoặc là dữ kiện dùng trong quá trình xử lý hay phục phụ cho bất  cứ nhu cầu nào phụ thuộc vào bussiness (nghiệp vụ) cài đặt.
- Websocket  là công nghệ hỗ trợ giao tiếp hai chiều giữa client và server bằng cách sử dụng một TCP socket để tạo một kết nối hiệu quả và ít tốn kém.

Action Cable cung cấp cơ chế để bạn đưa Rails app hoặc một phần nào của app có thể thực thi được tính năng realtime thông qua công nghệ WebSocket với phần hỗ trợ ở client là code Javascript và phần server là Ruby. Vì được tích hợp vào Rails nên bạn hoàn toàn có thể xử lý các truy vấn đến CSDL một cách dễ dàng thông qua ActiveRecord hoặc ORM khác.

Action Cable thông qua Rack socket để quản lý các kết nối tới server, đa luồng, tạo nên các kênh kết nối. Với mối kênh kết nối tới sub-URI của ứng dụng để truyền dữ liệu từ những vùng nhất định của dự án tới các vùng khác.

ActionCable cung cấp server-side code để broadcast nội dung nhất định ( new message hay notification) thông qua kênh "channel" tới một "subscriber". Subscriber này được khởi tạo từ phía clint-side với một hàm JS sử dụng JQuery để append nội dung vào DOM.

ActionCable sử dụng Redis để lưu trữ dữ liệu, đồng bộ nội dung thông qua các instances của ứng dụng. Sử dụng phương pháp Pub/Sub để giao tiếp giữa client và server.

### Pub/Sub:

Là mẫu gửi thông điệp mà người gửi (publishers), không lập trình thông điệp gửi trực tiếp tới người nhận cụ thể (subscribers). Thay vào đó, lập trình viên “gửi” các thông điệp (sự kiện), mà không hề biết gì về người nhận. Tương tự như vậy, người nhận thể hiện sở thích vào một hay nhiều sự kiện (events), và chỉ nhận thông điệp họ mong muốn, mà không hề biết về thông tin người gửi.

## Channel
Channel cung cấp những cấu trúc cơ bản để gom nhóm các hành vi thành các đơn vị logic khi giao tiếp thông qua kết nối Websocket

Channel object sẽ được tạo ra cho mỗi subscription mới cho mỗi kết nối

Action processing ở channel: không giống như subclasses của ActionController::Base, channel không tuân theo ràng buộc form về RESTful của action. Thay vào đó, Action Cable điều hành qua một remote-prodcedure. Ta có thể khai báo bất kỳ public method trên channel,  và method này sẽ có khả năng callable cho cilent

Channel sẽ bao gồm các thành phần sau:

### broadcasting.rb:

Truyền một hash đến một đường truyền đặc trưng cho model ở channel hiện tại
```ruby
def broadcast_to(model, message)
  ActionCable.server.broadcast(broadcasting_for([ channel_name, model ]), message)
end
```

### callback.rb:

Ta sẽ chú ý đến 2 method
- `on_subscribe`: Tên method được gọi khi channel được subscribe
```ruby
def on_subscribe(*methods)
  self.on_subscribe_callbacks += methods
end
```
- `on_unsubscribe`: Tên method được gọi khi channel được unsubscribe
```ruby
def on_unsubscribe(*methods)
  self.on_unsubscribe_callbacks += methods
end
```
Cả 2 method nên để private, để nó không thể gọi bởi user

### naming.rb:

Trả về tên của channel

Nếu channel là một namspace, thì namespace được đại diện bởi dấu phân cách ở tên channel
```ruby
def channel_name
  @channel_name ||= name.sub(/Channel$/, "").gsub("::", ":").underscore
end
```
VD:
+ ChatChannel.channel_name # => 'chat'
+ Chats::AppearancesChannel.channel_name # => 'chats:appearances'
+ FooChats::BarAppearancesChannel.channel_name # => 'foo_chats:bar_appearances'

### periodic_timers.rb:

Thể hiện công việc của channel, như cập nhật user truy cập trực tuyến, quyết định trạng thái mesages, v.v…

Truyền vào tên method hoặc lambda hoặc block để gọi Class này
```ruby
def periodically(callback_or_method_name = nil, every:, &block)
  callback =
    if block_given?
      raise ArgumentError, "Pass a block or provide a callback arg, not both" if callback_or_method_name
      block
    else
      case callback_or_method_name
      when Proc
        callback_or_method_name
      when Symbol
        -> { __send__ callback_or_method_name }
      else
        raise ArgumentError, "Expected a Symbol method name or a Proc, got #{callback_or_method_name.inspect}"
      end
    end

  unless every.kind_of?(Numeric) && every > 0
    raise ArgumentError, "Expected every: to be a positive number of seconds, got #{every.inspect}"
  end

  self.periodic_timers += [[ callback, every: every ]]
end
```

### streams.rb:

Cho phép các channel có thể broadcasting đến subscriber.

Đây là các method cần chú ý đến:
- `stream_from`: bắt đầu streaming từ danh sách broadcasting trong hàng đợi pubsub. Ta có thể bỏ qua một callback sẽ được thực hiện thay vì mặc định truyền các bản cập nhật đến subscriber.

Decode các mesages dạng JSON trước khi được callback.

Mặc định code: nil không được decode, mà sẽ truyền đi messages thô.
```ruby
def stream_from(broadcasting, callback = nil, coder: nil, &block)
  broadcasting = String(broadcasting)

  # Don't send the confirmation until pubsub#subscribe is successful
  defer_subscription_confirmation!

  # Build a stream handler by wrapping the user-provided callback with
  # a decoder or defaulting to a JSON-decoding retransmitter.
  handler = worker_pool_stream_handler(broadcasting, callback || block, coder: coder)
  streams << [ broadcasting, handler ]

  connection.server.event_loop.post do
    pubsub.subscribe(broadcasting, handler, lambda do
      ensure_confirmation_sent
      logger.info "#{self.class.name} is streaming from #{broadcasting}"
    end)
  end
end
```
- `stream_for`: bắt đầu streaming hàng đợi pubsub cho model của channel này
```ruby
def stream_for(model, callback = nil, coder: nil, &block)
  stream_from(broadcasting_for([ channel_name, model ]), callback || block, coder: coder)
end
```
- `stop_all_streams`: unsubscribes tất cả streams kết hợp với channel từ hàng đợi pubsub
```ruby
def stop_all_streams
  streams.each do |broadcasting, callback|
    pubsub.unsubscribe broadcasting, callback
    logger.info "#{self.class.name} stopped streaming from #{broadcasting}"
  end.clear
end
```

### Base.rb sẽ xử lý như sau:

- `perform_action`: xuất ra tên action từ dữ liệu được truyền và process thông qua channel. Các process sẽ được xác nhận là request action là một method công khai trên channel được khai báo bởi user.
```ruby
def perform_action(data)
  action = extract_action(data)

  if processable_action?(action)
    payload = { channel_class: self.class.name, action: action, data: data }
    ActiveSupport::Notifications.instrument("perform_action.action_cable", payload) do
      dispatch_action(action, data)
    end
  else
    logger.error "Unable to process #{action_signature(action, data)}"
  end
end
```
- `subscribe_to_channel`: method này được gọi sau khi subscription được thêm vào kết nối và được xác nhận hoặc từ chối subscription
```ruby
def subscribe_to_channel
  run_callbacks :subscribe do
    subscribed
  end

  reject_subscription if subscription_rejected?
  ensure_confirmation_sent
end
```
- `unsubscription_from_channel`: được gọi khi kết nối bị cắt, khi đó channel có khả năng bỏ hết callback. Method này không nên được gọi bởi user. Thay vào đó, nên dùng #unsubscribed callback
```ruby
def unsubscribe_from_channel # :nodoc:
  run_callbacks :unsubscribe do
    unsubscribed
  end
end
```
