# Vortext

Sane, componentized, forms in Rails.

If you've used Rails forms, or form helpers like simple_form, etc. you'll have run into a big mess.

Here's the base implementation:

```ruby
# frozen_string_literal: true

class FormComponent < ViewComponent::Base
  attr_reader :model, :url, :data

  renders_many :inputs, -> (field, as: :text, hint: nil, label: nil, **kwargs) do
    case as
    when :hidden
      InputComponent.new(model: model, field: field, type: :hidden)
    when :radio
      RadioButtonFieldComponent.new(model: model, field: field, type: as, hint: hint, label: label, **kwargs)
    else
      TextFieldComponent.new(model: model, field: field, type: as, hint: hint, label: label, **kwargs)
    end
  end

  renders_many :buttons, -> (type, value) do
    ButtonComponent.new(value: value, type: type)
  end

  def initialize(model:, url: nil, data:)
    @model = model
    @url = url || model
    @data = data
  end

  # Basic functionality for mapping a model attribute.
  class InputComponent < ViewComponent::Base
    attr_reader :type, :field, :model

    def initialize(model:, field:, type:)
      @model = model
      @field = field
      @type = type
    end

    def name
      helpers.field_name model.model_name.param_key, @field
    end

    def id
      "#{model.model_name.param_key}_#{@field}"
    end

    def value
      model.send field
    end
  end

  class HiddenFieldComponent < InputComponent
    def type
      :hidden
    end
  end

  # Handles more functionality such as the label for the field,
  # errors, and other information that an input would need that's
  # not hidden and is a textbox.
  class TextFieldComponent < InputComponent
    attr_reader :hint

    def initialize(hint: nil, label: nil, **kwargs)
      @label = label
      @hint = hint
      super(**kwargs)
    end

    def label
      @label ||= field.to_s.humanize
    end

    def errors
      model.errors[field]
    end

    def error_message
      "#{label} #{model.errors.generate_message(field)}"
    end

    def invalid?
      errors.any?
    end

    def error_classes
      "border-red-500"
    end

    def valid_classes
      ""
    end

    def classes
      invalid? ? error_classes : valid_classes
    end
  end

  class ButtonComponent < ViewComponent::Base
    attr_reader :value, :type

    def initialize(value:, type:)
      @value = value
      @type = type
    end
  end

  class CollectionComponent < TextFieldComponent
    attr_reader :collection

    def initialize(collection: [], **kwargs)
      super(**kwargs)
      @collection = normalize_collection collection
    end

    private
      def normalize_collection(collection)
        case collection
        when Hash
          collection.to_a
        else
          collection
        end
      end
  end

  class RadioButtonFieldComponent < CollectionComponent
    def type
      :radio
    end
  end
end

```

Consuming it looks like this: 

```slim
= article_component title: "Email subscription", subtitle: "Get Legible News in your email inbox" do
  = form_component model: @email_subscription, url: url_for(action: :create), data: { turbo: false } do |f|
    .my-8
      = f.input :email, as: :email, hint: "We'll deliver issues to this email address"
    .my-8
      = f.input :edition, as: :radio, hint: "How often would you like the news in your inbox?", collection: EmailSubscription::EDITIONS
    .my-8
      = f.button :submit, "Save email subscription settings"
```

You get to build the intent of your form in a view. Then each field is wrapped in a component, which you can customize by subclasses.

Here's the structure of the view files. You have it all in one place.

<img width="373" alt="Screen Shot 2022-06-07 at 00 14 38" src="https://user-images.githubusercontent.com/4628/172319112-64d83c01-e955-430a-877e-7c8048c4aee7.png">

The awesome part is how you can put whatever in each form field component. It's like this for a text field:

```slim
label.block.text-gray-700.dark:text-gray-200.font-bold.mb-2 for=id
  = label

input.shadow.appearance-none.border.rounded.w-full.py-2.px-3.text-gray-700.mb-1.leading-tight.focus:outline-none.focus:shadow-outline.dark:bg-neutral-800.dark:text-neutral-200 class=classes type=type value=value name=name id=id /

-if invalid?
  p.text-red-600.text-sm.my-1=error_message

-if hint
  p.hint= hint
```

It's a lot, but you get super fine grain control of your tailwindcss utilitly libraries.
