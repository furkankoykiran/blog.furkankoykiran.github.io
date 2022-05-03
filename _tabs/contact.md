---
title: İletişim
icon: far fa-address-book
layout: page
order: 5
---


> **Not**: Aşağıdaki form üzerinden benimle iletişime geçebilirsiniz.

<form id="my-form" method="POST" action="https://formspree.io/f/mzbokoyz">
  <input type="email" name="email" class="contact" placeholder="E Posta Adresiniz">
  <input type="text" name="name" class="contact" placeholder="Adınız">
  <textarea name="message" class="contact" placeholder="Mesajınız" rows="3">
  </textarea>
  <br>
  <button type="submit" id="my-form-button" class="contact">Gönder</button>
  <p id="my-form-status" class="contact"></p>
</form>

<!-- Place this script at the end of the body tag -->

<script>
    var form = document.getElementById("my-form");
    
    async function handleSubmit(event) {
      event.preventDefault();
      var status = document.getElementById("my-form-status");
      var data = new FormData(event.target);
      fetch(event.target.action, {
        method: form.method,
        body: data,
        headers: {
            'Accept': 'application/json'
        }
      }).then(response => {
        status.innerHTML = "Gönderdiğiniz için teşekkürler!";
        form.reset()
      }).catch(error => {
        status.innerHTML = "Oops! Formunuzu gönderirken bir problem oluştu"
      });
    }
    form.addEventListener("submit", handleSubmit)
</script>