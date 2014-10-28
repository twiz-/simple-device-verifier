simple-device-verifier
======================

I wanted to simply verify that a user had access to the actual phonenumber they were entering before saving the record to my rails backend. Here is the implementation-

Here is what you should have by the end of it-

Enter in a phone number-

![alt text](http://fat.gfycat.com/LegitimateEvenFoal.gif "enter number")

and get a verification code to save record-
![alt text](http://fat.gfycat.com/ReflectingTheseAfricanparadiseflycatcher.gif "enter number")

I have a form where a user inputs any phone numer and submits the form. I have this form responding to AJAX that prompts the user to then enter in the verification they receive. I'm sending the number as `:unverified` that I'll reference later. This form submits to an action in my controller called `send_verification`

```ruby
<%= form_tag("/route-endpoint/", method: "post", remote: true, role: "form", id: "phonenumberForm") do %>
<div class="form-group">
<label>Enter phone number with</label>
  <%= text_field_tag(:unverified_number,nil, class: "form-control", placeholder: "ex: +14151234567", data: {"bv-phone-message" => true} )%>
</div>
<div class="form-group">
  <%= button_tag("Send Number",class: "btn btn-block btn-success btn-lg") %>
  <small><i>By filling out this form you agree to the <%= link_to "terms", terms_path %></i></small>
</div> 
<% end %> 
```

Here is the `send_verification` method in my controller. There are a couple ways to do this but I'm storing the data sent by the user in a flash variable. This is cool because the value stored here is only avialable in the next request and we only care about it for a brief moment in time.
The flash variable is typically used for things like notices, confirmations, or errors messages but in this case we're using it to store an `:unverified_number` and a `:verify_code`.

Then I'm setting up a Twilio client to send an sms to the phone number entered along with a randomly generated 6 digit number. `send_verification_code_for_signup` is a method in my `TwilioWorker` that just sends an sms to a number with my credentials. 

Finally, I'm responding with a form that will actually create and save a verified phonenumber as a record. 

```ruby

  def send_verification
    flash[:number] = params[:unverified_number]
    flash[:verify_code] = SecureRandom.random_number.to_s[-6..-1]
    
    client = TwilioWorker.new(flash[:number],flash[:verify_code])
    client.send_verification_code_for_signup    
    respond_to do |format|
      format.js
    end
  end
  
```
3 things happen after this action is triggered
- an sms is sent to visitor
- the first form disappears
- a new form appears to enter in the sms code the visitor was just sent


So, now in my `send_verification.js.erb` file I have a form that asks for only a verfication code. 

```ruby
<%= form_tag("phone number create action", method: "post") do %>
<%= text_field_tag(:verify_code, nil, class: "form-control", placeholder: "ex: 85666")%>
<%= button_tag("Verifify My phone number",class: "btn btn-block btn-warning btn-lg") %>
<% end %>
```

This form creates a Phone Number record and checks the verify code matches the flash variable from before. Here is what the create action this form submits to looks like. 

We set the phonenumber equal to what was sent in the first form via the flash variable. This simply says don't save the phonenumber unless the verification code sent in matches the verification code set to the flash variable earlier. 

```ruby

def create
PhoneNumber.create(number: flash[:number], matcher: params[:verify_code]) unless params[:verify_code] != flash[:verify_code]    
end

```

I'm saving the verify code to the number because I want to test out strategies to not send the same verify code twice (just as an exercise), but you could conceivably simplify it further- 

```ruby
def create
PhoneNumber.create(number: flash[:number]) unless params[:verify_code] != flash[:verify_code]    
end
```

You'll need to define some additional logic if the number is successfully saved or not but that's pretty much it! I know this strategy can be improved and may have some faults but it worked for my purposes at this moment in time. One js library I'm using to help ensure a phone number is a phone number is http://bootstrapvalidator.com/ which makes it easy to do client side validations. 

Open issue for improvements or flaws in this strategy. 
