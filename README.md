const sgMail = require('@sendgrid/mail');
const twilio = require('twilio');
const db = require('../config/database');

/**
 * Automated Customer Communication Service
 * Handles all order status updates via email, SMS, and WhatsApp
 */

class CustomerCommunicationService {
  constructor() {
    sgMail.setApiKey(process.env.SENDGRID_API_KEY);
    this.twilio = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);
    
    // Email templates for each order status
    this.templates = {
      orderReceived: 'd-order-received',
      paymentConfirmed: 'd-payment-confirmed',
      fnoSubmitted: 'd-fno-submitted',
      installationScheduled: 'd-installation-scheduled',
      technicianAssigned: 'd-technician-assigned',
      installationComplete: 'd-installation-complete',
      activationPending: 'd-activation-pending',
      serviceActive: 'd-service-active',
      orderCompleted: 'd-order-completed',
      delayNotification: 'd-delay-notification',
      documentRequired: 'd-document-required'
    };
  }

  // Main method: Send status update
  async sendStatusUpdate(orderId, status, additionalData = {}) {
    try {
      // Get order with customer details
      const order = await this.getOrderWithCustomer(orderId);
      if (!order) throw new Error('Order not found');

      // Check POPI consent for communications
      const hasConsent = await this.checkCommunicationConsent(order.customer.id);
      if (!hasConsent) {
        console.log(`No communication consent for customer ${order.customer.id}`);
        return { sent: false, reason: 'no_consent' };
      }

      // Prepare message data
      const messageData = this.prepareMessageData(order, status, additionalData);

      // Send via all configured channels
      const results = await Promise.allSettled([
        this.sendEmail(order.customer.email, status, messageData),
        this.sendSMS(order.customer.phone, status, messageData),
        this.sendWhatsApp(order.customer.phone, status, messageData)
      ]);

      // Log communications
      await this.logCommunication(orderId, status, results);

      return {
        sent: true,
        channels: {
          email: results[0].status === 'fulfilled',
          sms: results[1].status === 'fulfilled',
          whatsapp: results[2].status === 'fulfilled'
        }
      };

    } catch (error) {
      console.error('Status update failed:', error);
      throw error;
    }
  }

  async sendEmail(email, status, data) {
    const template = this.templates[status] || this.templates.orderReceived;
    
    const msg = {
      to: email,
      from: {
        email: 'orders@connectpro.co.za',
        name: 'ConnectPro Orders'
      },
      templateId: template,
      dynamicTemplateData: {
        customerName: data.customer.firstName,
        orderReference: data.order.reference,
        orderDate: new Date(data.order.createdAt).toLocaleDateString('en-ZA'),
        ispName: data.isp.name,
        productName: data.product.name,
        monthlyCost: data.product.price,
        status: this.formatStatus(status),
        nextSteps: data.nextSteps,
        trackingUrl: `${process.env.APP_URL}/track/${data.order.reference}`,
        supportPhone: '087 123 4567',
        supportEmail: 'support@connectpro.co.za'
      },
      categories: ['order_status', status, data.isp.key]
    };

    await sgMail.send(msg);
    return { channel: 'email', sent: true };
  }

  async sendSMS(phone, status, data) {
    const messages = {
      orderReceived: `Hi ${data.customer.firstName}, your order ${data.order.reference} with ${data.isp.name} has been received. Track at ${process.env.APP_URL}/track/${data.order.reference}`,
      installationScheduled: `Great news! Your ${data.isp.name} installation is scheduled for ${data.scheduledDate}. Please ensure someone is available.`,
      serviceActive: `Your ${data.isp.name} service is now active! Login details sent to your email. Welcome to ConnectPro!`
    };

    const message = messages[status] || `Update on order ${data.order.reference}: ${this.formatStatus(status)}`;

    await this.twilio.messages.create({
      body: message,
      from: process.env.TWILIO_PHONE_NUMBER,
      to: this.formatPhone(phone)
    });

    return { channel: 'sms', sent: true };
  }

  async sendWhatsApp(phone, status, data) {
    // WhatsApp Business API integration
    const message = {
      messaging_product: "whatsapp",
      to: this.formatPhone(phone),
      type: "template",
      template: {
        name: `order_${status}`,
        language: { code: "en" },
        components: [
          {
            type: "body",
            parameters: [
              { type: "text", text: data.customer.firstName },
              { type: "text", text: data.order.reference },
              { type: "text", text: data.isp.name }
            ]
          }
        ]
      }
    };

    // Call WhatsApp API
    await this.callWhatsAppAPI(message);
    return { channel: 'whatsapp', sent: true };
  }

  // Triggered by order status changes
  async handleOrderStatusChange(orderId, oldStatus, newStatus, metadata = {}) {
    const statusFlow = {
      'draft': { next: 'submitted', template: 'orderReceived' },
      'submitted': { next: 'payment_pending', template: null },
      'payment_received': { next: 'fno_submitted', template: 'fnoSubmitted' },
      'fno_acknowledged': { next: 'installation_scheduling', template: null },
      'installation_scheduled': { next: 'installation_pending', template: 'installationScheduled' },
      'technician_assigned': { next: 'installation_in_progress', template: 'technicianAssigned' },
      'installation_complete': { next: 'activation_pending', template: 'installationComplete' },
      'service_activated': { next: 'active', template: 'serviceActive' },
      'completed': { next: null, template: 'orderCompleted' }
    };

    const config = statusFlow[newStatus];
    if (!config) return;

    // Send notification
    if (config.template) {
      await this.sendStatusUpdate(orderId, config.template, metadata);
    }

    // Schedule follow-up communications
    await this.scheduleFollowUps(orderId, newStatus, metadata);
  }

  // Schedule automated follow-up messages
  async scheduleFollowUps(orderId, currentStatus, metadata) {
    const schedules = {
      'installation_scheduled': [
        { delay: 24 * 60 * 60 * 1000, type: 'reminder', message: 'Installation tomorrow reminder' },
        { delay: 2 * 60 * 60 * 1000, type: 'final_reminder', message: 'Installation in 2 hours' }
      ],
      'service_activated': [
        { delay: 7 * 24 * 60 * 60 *
