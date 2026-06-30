**De:** Gustavo Rodrigues  
**Para:** Weber Muniz do Carmo  
**Cc:** Samuel Gonçalves Pereira  
**Assunto:** PoC ProFarma — Conclusão e próximos passos

---

Weber, boa tarde!

Concluímos a Prova de Conceito que havíamos alinhado e gostaria de compartilhar um resumo do que foi feito e deixar as portas abertas para o que você precisar.

### O que entregamos

- **Ambiente Rundeck + AWX** rodando lado a lado no mesmo cluster, via Kustomize — deploy reprodutível com um comando (`kubectl apply -k`).
- **Playbook real** executado contra o servidor Windows da ProFarma — pipeline de 4 etapas com geração de arquivos, cópia, validação de checksum e relatório consolidado.
- **Comparação Rundeck vs AWX** — mesma automação, dois ambientes, para você avaliar qual entrega melhor visibilidade, orquestração e dashboard para o seu caso.
- **Documentação completa** — guia passo a passo, apresentação com comparativo técnico e recomendações.

Todo o material está disponível no repositório que compartilhamos com você.

### Perguntas?

Ficou alguma dúvida sobre o que foi apresentado? Algum ponto que gostaria que a gente aprofundasse ou repetisse?

### Apresentação para gestores

Se fizer sentido aí, podemos agendar uma apresentação para o seu time de gestão — mostrando o racional da escolha, o comparativo entre as ferramentas e o roadmap de implementação. É algo rápido, 30 a 40 minutos, e ajuda a alinhar as expectativas com quem decide.

### Próximos passos

Quando vocês estiverem prontos para avançar, sugerimos:

1. **Definir os 10 workflows prioritários** para a fase 1 de implementação
2. **Alinhar acessos definitivos** (CyberArk) para o ambiente de produção
3. **Agendar o kickoff** com o time técnico que vai operar as ferramentas

Estamos à disposição para tirar qualquer dúvida. Pode me chamar aqui no e-mail ou no WhatsApp — o que for mais prático.

Grande abraço,

**Gustavo Rodrigues**  
Consultoria 4Linux  

**Samuel Gonçalves Pereira**  
Gerente de Tecnologia — 4Linux
